



# ReleapIO_sui-nft-protocol

被扫描到的结构体：

```
/// Collection generic witness type
struct Witness<phantom C> has copy, drop {}
```

`from_witness<C, W>(_witness: &W)`: 从一个已有的、代表集合 `C` 权限的“原始”见证者 `W`（通过 `assert_same_module_as_witness` 验证）来创建一个 `Witness<C>`。

`delegate<C>(_generator: &WitnessGenerator<C>)`: 通过一个 `WitnessGenerator<C>`（这个生成器本身可能也是由原始见证者 `W` 创建的）来创建 `Witness<C>`。

```
/// Module of the `WitnessGenerator` used for generating authenticating
/// witnesses on demand.
module nft_protocol::witness {
    use nft_protocol::utils;

    /// Collection witness generator
    struct WitnessGenerator<phantom C> has store {}

    /// Collection generic witness type
    struct Witness<phantom C> has copy, drop {}

    /// Create a new `WitnessGenerator` from collection witness
    public fun generator<C, W>(_witness: &W): WitnessGenerator<C> {
        utils::assert_same_module_as_witness<C, W>();
        WitnessGenerator {}
    }

    /// Delegate a witness from collection witness
    public fun from_witness<C, W>(_witness: &W): Witness<C> {
        utils::assert_same_module_as_witness<C, W>();
        Witness {}
    }

    /// Delegate a collection generic witness
    public fun delegate<C>(_generator: &WitnessGenerator<C>): Witness<C> {
        Witness {}
    }
}

```

基于其名称 "Witness" 以及它是从一个原始见证者 `W` 派生而来的事实，最合理的推断是 `Witness<C>` 被设计用来在代码的其他地方证明调用者拥有对集合 `C` 的某种权限（例如，铸造 NFT 到该集合的权限）。

如果 `Witness<C>` 确实如其名称和产生方式所暗示的那样，是用作下游函数中对集合 `C` 相关操作的授权凭证，那么 `copy` 能力的存在就**直接违反了**标准见证者模式的核心安全原则。



