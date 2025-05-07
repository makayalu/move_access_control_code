## Aptos

### 1.可变引用（*）

返回值为可变引用的函数应为私有函数。

```rust
// aptos move
public fun get_mut_rollback(self: &mut Storage, sequence_no: u128): &mut RollbackData {
    table::borrow_mut(&mut self.rollbacks, sequence_no)
}
```

后果：公开暴露可变引用会破坏封装性，允许外部代码绕过业务逻辑直接修改数据。`RollbackData`若包含关键交易回滚信息，外部直接修改可能导致回滚逻辑失效，甚至引发资金损失。

方法：检查函数可见性和返回可变引用组合。

### 2.对象所有权（*）

对象类别(sui)

- Address-Owned Objects：由特定地址（账户地址或对象 ID）拥有的对象，只有对象的所有者可以访问和操作这些对象。
- Immutable Objects：不可变对象无法被修改或转移，任何人都可以访问。适用于需要全局访问但不需要修改的数据。
- Shared Objects：共享对象可以被多个用户访问和操作，适用于去中心化应用等场景，但由于需要共识，操作成本较高。
- Wrapped Objects：包装对象是将一个对象嵌入另一个对象，包装后的对象不再独立存在，必须通过包装对象访问。

sui中uid是唯一的，id不是唯一的

aptos与sui的对象不同，任何人都可以访问一个Object，需要主动检验权限

```rust
// aptos move
entry fun execute_action_with_valid_subscription(
        user: &signer, obj: Object<Subscription>
    ) acquires Subscription {
        //ensure that the signer owns the object.
        assert!(object::owner(&obj)==address_of(user),ENOT_OWNWER);
```

后果：参考链接：https://aptos.dev/en/build/smart-contracts/object/using-objects#looking-up-who-owns-an-object。

aptos中的对象可以由任何地址拥有，因此验证所有权需要考虑所有者。在一个对象创建之后，可以直接通过object的地址传入函数中（不再需要原生move的borrow_global等操作）。

在实际项目中，协议通常各种资产和信息都放在一个object上方便管理，例如下面PositionAssetStore对象包含coin和剩余保证金信息，如果不验证所有者的话任何人都可以提取别人仓位下的代币。

方法：1.传参包含，需要验证object在一个函数的上下文中是否有修改（包括object::transfer）2.传参是否传递caller相关信息（主要是signer）3.判断条件：owner与signer地址比较（Aptos 提供了诸如 `object::owner`、`object::is_owner` 和 `object::owns` 等函数来检查对象的所有者是否为指定地址） 4.可能存在所有权嵌套关系，所以可能会向下追溯~~（做实验时这点可以先不做）~~ 5.可能需要跨模块寻找地址比较，因为比较函数有可能不在当前模块中（基本不考虑）

难点：这是aptos特有的，需要考虑在检测时要不要判断是什么链的合约。如果要判断，可以通过找object里面或者顶层object是否包含uid。~~（应该不需要，这是aptos特有的编程patten，单纯检测即可，并且在现实项目漏洞检测中，人为确定是什么项目匹配什么样的检测器就好）~~（不一定可行，因为这涉及move依赖的问题）

注意点：由于取出所需object的方法是多样的，所有权判断的函数也是多样的，所以要明确哪些函数是会涉及到的，防止漏报。因而重点阅读aptos的object库，account和signer库。

![image-20250224182726178](C:/Users/Tony/Desktop/下一篇/move相关/move/move_question/image-20250224182726178.png)

![image-20250224182029503](C:/Users/Tony/Desktop/下一篇/move相关/move/move_question/image-20250224182029503.png)

### 全局存储访问控制-同上

此代码片段允许任何用户调用该`delete`函数来删除`Object`，而无需验证调用者是否具有必要的权限。

```rust
module 0x42::example {
  struct Object has key{
    data: vector<u8>
  }
 
  public fun delete(user: &signer, obj: Object) {
    let Object { data } = obj;
  }
}
```

![image-20250224182937023](C:/Users/Tony/Desktop/下一篇/move相关/move/move_question/image-20250224182937023.png)

缓解方案：更好的替代方案是使用 Move 提供的全局存储，直接从 借用数据`signer::address_of(signer)`。这种方法可确保强大的访问控制，因为它仅访问交易签名者地址中包含的数据。这种方法最大限度地降低了访问控制错误的风险，确保只有拥有的数据`signer`才能被操纵。

```rust
module 0x42::example {
  struct Object has key{
    data: vector<u8>
  }
  public fun delete(user: &signer) {
    let Object { data } = move_from<Object>(signer::address_of(user));
  }
}

// 修改
  public fun manipulate(user: &signer) acquires Object {
    let ref: &mut Object = borrow_global_mut<Object>(signer::address_of(user));
}
```

### 3.构造函数泄漏（*）

不要返回CostructorRef。

```rust
// aptos move
module 0x42::example {
  use std::string::utf8;
 
  public fun mint(creator: &signer): ConstructorRef {
    let constructor_ref = token::create_named_token(
        creator,
        string::utf8(b"Collection Name"),
        string::utf8(b"Collection Description"),
        string::utf8(b"Token"),
        option::none(),
        string::utf8(b"https://mycollection/token.jpeg"),
    );
    constructor_ref
  }
}
```

后果：例如，如果某个`mint`函数返回`ConstructorRef`NFT 的 ，则可以将其转换为`TransferRef`，存储在全局存储中，并允许原来所有者在 NFT 出售后将其转回。

方法：检测返回值是否有ConstructorRef

aptos官方提到了这个问题，参考链接：https://aptos.dev/en/build/smart-contracts/move-security-guidelines#constructorref-leak。

### 4.对象账户-对象隔离（*）

在 Aptos 框架中，多个`key`可存储资源可以存储在单个对象帐户中。

但是，对象应该被隔离到不同的账户，否则对账户内一个对象的修改可能会影响整个集合。

例如，转移一种资源意味着转移所有组成员，因为转移函数作用于`ObjectCore`，它本质上是账户中所有资源的通用标签。

由于`Monkey`和`Toad`属于同一对象帐户，因此两个对象现在都归所有`recipient`。

```rust
// aptos move
module 0x42::example {
  #[resource_group(scope = global)]
  struct ObjectGroup { }
 
  #[resource_group_member(group = 0x42::example::ObjectGroup)]
  struct Monkey has store, key { }
 
  #[resource_group_member(group = 0x42::example::ObjectGroup)]
  struct Toad has store, key { }
 
  fun mint_two(sender: &signer, recipient: &signer) {
    // sender->caller
    // object(constructor_ref)
    let constructor_ref = &object::create_object_from_account(sender);
    // object->object_signer
    let sender_object_signer = object::generate_signer(constructor_ref);
    // object_addr
    let sender_object_addr = object::address_from_constructor_ref(constructor_ref);
 
    // caller-object-monkey+toad  
    move_to(sender_object_signer, Monkey{});
    move_to(sender_object_signer, Toad{});
    // resource1->object1 
    let monkey_object: Object<Monkey> = object::address_to_object<Monkey>(sender_object_addr);
    object::transfer<Monkey>(sender, monkey_object, signer::address_of(recipient));
  }
}
```

应该分开存储

```rust
module 0x42::example {
  #[resource_group(scope = global)]
  struct ObjectGroup { }
 
  #[resource_group_member(group = 0x42::example::ObjectGroup)]
  struct Monkey has store, key { }
 
  #[resource_group_member(group = 0x42::example::ObjectGroup)]
  struct Toad has store, key { }
 
  fun mint_two(sender: &signer, recipient: &signer) {
    let sender_address = signer::address_of(sender);
 
    let constructor_ref_monkey = &object::create_object(sender_address);
    let constructor_ref_toad = &object::create_object(sender_address);
    let object_signer_monkey = object::generate_signer(&constructor_ref_monkey);
    let object_signer_toad = object::generate_signer(&constructor_ref_toad);
 
    move_to(object_signer_monkey, Monkey{});
    move_to(object_signer_toad, Toad{});
 
    let object_address_monkey = signer::address_of(&object_signer_monkey);
 
    let monkey_object: Object<Monkey> = object::address_to_object<Monkey>(object_address_monkey);
    object::transfer<Monkey>(sender, monkey_object, signer::address_of(recipient));
  }
}
```

方法：（1）是否存在多个resource在一个object对象下（2）需要对单个对象账户下的资源转移（或修改）（3）是否修改会影响Object的内容，需要测试（4）权限全部转移到一个object中？

难点：1.就是想多个resource托管在同一个object 2.需要测一下是否如描述的结果（对象应该被隔离到不同的账户，否则对账户内一个对象的修改可能会影响整个集合。）

### 5.随机性 - test-and-abort（*）

Tip：在Aptos，我们始终是安全优先。 在汇编过程中，我们确保没有从public函数中调用随机性API。 但是，我们仍然允许用户通过添加属性```＃[lint :: lind_unsafe_randomness]```来做出此选择。

如果public函数直接或间接调用随机性API，则恶意用户可以滥用此功能的合成性，并在结果不尽可能地要求的情况下中止交易。 这使用户能够继续尝试，直到获得有益的结果，从而破坏随机性。

```rust
module user::lottery {
    fun mint_to_user(user: &signer) {
        move_to(user, WIN {});
    }
 
    #[lint::allow_unsafe_randomness]
    public entry fun play(user: &signer) {
        let random_value = aptos_framework::randomness::u64_range(0, 100);
        if (random_value == 42) {
            mint_to_user(user);
        }
    }
}
```

方法：函数是public，并且调用了randomness API。

### 6.随机性 - undergasing（*）

当功能中不同的代码路径消耗不同量的气体时，攻击者可以操纵气体限制以使结果偏差。

```rust
module user::lottery {
 
    //transfer 10 aptos from admin to user
    fun win(user: &signer) {
        let admin_signer = &get_admin_signer();
        let aptos_metadata = get_aptos_metadata();
        primary_fungible_store::transfer(admin_signer, aptos_metadata, address_of(user),10);
    }
 
    //transfer 10 aptos from user to admin, then 1 aptos from admin to fee_admin
    fun lose(user: &signer) {
 
        //user to admin
        let aptos_metadata = get_aptos_metadata();
        primary_fungible_store::transfer(user, aptos_metadata, @admin, 10);
 
        //admin to fee_admin
        let admin_signer = &get_admin_signer();
        primary_fungible_store::transfer(admin_signer, aptos_metadata, @fee_admin, 1);
    }
 
    #[randomness]
    entry fun play(user: &signer) {
        let random_value = aptos_framework::randomness::u64_range(0, 100);
        if (random_value == 42) {
            win(user);
        } else {
            lose(user);
        }
    }
}
```

方法：函数中调用randomness，并存在分支调用。

## Sui

### 1.Witness带有copy

在 Move 中，资源（Resource）是拥有独特所有权和生命周期的实体。出于安全考虑，某些资源不希望由外部随意构造或复制。见证者模式通过引入一个“见证者”参数——一般是一个只有 drop 能力的资源（这并不绝对）——来证明调用者确实拥有创建这些资源的权限。只有提供了正确的见证者，才能调用对应函数生成资源。

```rust
// 有drop，意味着不需要显示析构witness
module book::one_time {
    /// The OTW for the `book::one_time` module.
    /// Only `drop`, no fields, no generics, all uppercase.
    public struct ONE_TIME has drop {}

    /// Receive the instance of `ONE_TIME` as the first argument.
    fun init(otw: ONE_TIME, ctx: &mut TxContext) {
        // do something with the OTW
    }
}

// 无drop
module book::one_time {
    /// The OTW for the `book::one_time` module.
    /// Only `drop`, no fields, no generics, all uppercase.
    public struct ONE_TIME {}

    /// Receive the instance of `ONE_TIME` as the first argument.
    fun init(otw: ONE_TIME, ctx: &mut TxContext) {
        let ONE_TIME {} = otw;
        // do something with the OTW
    }
}
```

见证者模式在 Move 智能合约中扮演着至关重要的角色，它通过强制要求在特定操作中提供一个只能被消耗一次的见证者资源，从而实现受控实例化和权限管理。所以见证者不能包含copy能力

```rust
module movebit::witness {
    // A Witness type with copy ability. This often causes trouble
    use std::vector;
    struct Witness has copy {}

    public fun get_witness(): Witness {
        return Witness{}
    }

    public fun access_via_witness(witness: Witness) {
        // destroy Witness
        let Witness {} = witness;
        let v = vector::empty<u64>();
        vector::push_back(&mut v, 5);
        // something can only be done once controlled by witness.
    }
}



module movebit::witness_user {
    use movebit::witness;

    fun test() {
        let witness = witness::get_witness();
        let copied_witness = witness; // DETECT: give out a warning msg to prompt Witness is copied.
        witness::access_via_witness(witness);
        witness::access_via_witness(copied_witness)
    }
}
```

方法：1.struct不含任何内容一般就是witness 2.检测它是否具有copy 3.后面真的静态检测还需要考虑一些东西

预实验：在aptos上面也扫一下看看

注意点：不含内容的struct其实可能还在hot potato模式里使用，可能会有误报
