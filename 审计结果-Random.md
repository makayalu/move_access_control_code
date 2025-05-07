### 项目1 ArielRobotti_AptosBlackJack

```
    public entry fun iniciarPartida(cuenta: &signer) {
        let mazo = shuffle_and_cut_deck();
        move_to(cuenta, mazo)
    }
    
    fun shuffle_and_cut_deck(): Deck {

            let sortedDeck = vector<u8>[
                1,2,3,4,5,6,7,8,9,10,10,10,10,
                1,2,3,4,5,6,7,8,9,10,10,10,10,
                1,2,3,4,5,6,7,8,9,10,10,10,10,
                1,2,3,4,5,6,7,8,9,10,10,10,10,
            ];
            let unsortedDeck = empty<u8>();
            for (i in 0..52) {
                let currentSize = length(& sortedDeck) ;
                let index = aptos_framework::randomness::u64_range(0, currentSize);
                let selectedCard = remove(&mut sortedDeck, index);
                push_back(&mut unsortedDeck, selectedCard);
            };

            let cut_card = aptos_framework::randomness::u8_range(0, 13) % 13 + 26; // colocamos el naipe de corte entre la posicion 26 y la 39
            Deck {deck: unsortedDeck, cut_card}
        }
```

**存在漏洞**。用户调用 `public entry fun iniciarPartida`。

`iniciarPartida` 内部调用 `shuffle_and_cut_deck`。

`shuffle_and_cut_deck` 内部调用 `aptos_framework::randomness::u64_range` 和 `aptos_framework::randomness::u8_range` 来生成随机数，决定牌堆顺序和切牌点。

用户根据这个初始牌堆状态判断开局对自己不利（例如，前几张牌很差，或者切牌位置不理想），他们可以选择让交易失败或不支付足够的 gas 使其失败。

### 项目2 ayushchamp_Aptos-Project

```
    #[randomness]
    entry fun randomly_set_computer_move(account: &signer) acquires Game {
        randomly_set_computer_move_internal(account);
    }
    
    public(friend) fun randomly_set_computer_move_internal(account: &signer) acquires Game {
        let game = borrow_global_mut<Game>(signer::address_of(account));
        let random_number = randomness::u8_range(1, 4);
        game.computer_move = random_number;
    }
```

**存在漏洞**。用户调用 `entry fun randomly_set_computer_move`。

`randomly_set_computer_move` 内部调用 `randomly_set_computer_move_internal`。

`randomly_set_computer_move_internal` 调用 `randomness::u8_range` 来获取一个随机数作为计算机的出拳。

由于入口函数 `randomly_set_computer_move` 间接调用了随机性 API，并且没有标记 `#[lint::allow_unsafe_randomness]`，这构成了“随机性滥用”漏洞。

### 项目3 caoyang2002_aptos_mvoe-learning

```
    entry fun roll_v0(_account: signer) {
        let _ = randomness::u64_range(0, 6);
    }
    
    #[randomness(max_gas=56789)]
    entry fun roll_v2(_account: signer) {
        let _ = randomness::u64_range(0, 6);
    }

    #[randomness]
    /// Can only be called as a top-level call from a TXN, preventing **test-and-abort** attacks (see
    /// [AIP-41](https://github.com/aptos-foundation/AIPs/blob/main/aips/aip-41.md)).
    entry fun randomly_pick_winner() acquires Raffle {
        randomly_pick_winner_internal();
    }
```

**存在漏洞**。`dice::roll_v0`、`dice::roll_v2` 和 `raffle::randomly_pick_winner`中存在的随机性滥用漏洞。

### 项目4 CopperRoger_count-fi

**误报**，合约未使用 `aptos_framework::randomness` API，而是采用了基于时间戳的（同样不安全的）伪随机性方法。

### 项目5 gregnazario_aptos-hybrid-assets

```
    entry fun reveal(caller: &signer, token: Object<HybridToken>) acquires ExampleHybrid, RevealStageMetadata {
        reveal_inner(caller, token);
    }
    
    entry fun reveal_many(caller: &signer, tokens: vector<address>) acquires ExampleHybrid, RevealStageMetadata {
        vector::for_each(tokens, |token| {
            let token_object = object::address_to_object(token);
            reveal_inner(caller, token_object);
        })
    }

 fun reveal_inner(caller: &signer, token: Object<HybridToken>) acquires ExampleHybrid, RevealStageMetadata {
        let caller_address = signer::address_of(caller);
        assert!(object::is_owner(token, caller_address), E_NOT_OWNER_TOKEN);

        let collection_object = token::collection_object(token);
        let collection_address = object::object_address(&collection_object);

        let example = borrow_global<ExampleHybrid>(collection_address);
        let reveal_data = borrow_global<RevealStageMetadata>(collection_address);

        // This scheme, is to randomly choose ANY of the possible outcomes.  This does not prevent against duplicates.
        let roll_number = randomness::u64_range(0, MAX_COMBINATIONS);
        let name = reveal_data.name_prefix;
        string::append_utf8(
            &mut name,
            *string::bytes(&string_utils::to_string_with_integer_types(&roll_number))
        );

        let uri = reveal_data.nft_base_uri;
        string::append(&mut uri, string_utils::to_string(&roll_number));
        string::append(&mut uri, reveal_data.nft_uri_extension);

        hybrid::reveal(
            &example.reveal_ref,
            token,
            option::some(name),
            option::none(),
            option::some(uri),
            false,
        );
    }
```

**存在漏洞**。核心的、对用户开放的 **`reveal` 和 `reveal_many` 函数存在明确的随机性滥用漏洞**，因为它们是 `entry fun`，间接调用了 `randomness::u64_range`，并且没有使用 `#[lint::allow_unsafe_randomness]` 属性。这允许用户通过重试来操纵 NFT 揭示的结果。

### 项目6 guinjgida_mint_nft_aptos

```

    public fun buy_cat(sender:&signer,nft_address:address) acquires OnChainConfig{
        let config = borrow_global_mut<OnChainConfig>(nft_address);
        assert!(config.status,ENOT_START);

        let coins = coin::withdraw<AptosCoin>(sender,config.price);
        coin::deposit(config.payee,coins);
        create(sender,config);
        config.index = config.index + 1;
    }
    
    entry fun buy_cat_entry(
        account: &signer,
        nft_address: address
    ) acquires OnChainConfig {
        buy_cat(account, nft_address);
    }

    fun create(creator:&signer,onchain_config:&OnChainConfig){
            let resource_signer = account::create_signer_with_capability(&onchain_config.signer_cap);
            let random_num = randomness::u64_range(0,2);
            let url = *vector::borrow(&onchain_config.image_urls,random_num);
            let name = get_token_name(onchain_config.name,onchain_config.index + 1);
            let token_object = aptos_token::mint_token_object(
                &resource_signer,
                onchain_config.collection,
                onchain_config.description,
                name,
                url,
                vector<String>[],
                vector<String>[],
                vector<vector<u8>>[],
            );
            object::transfer(&resource_signer,token_object,signer::address_of(creator));
        }
```

**存在漏洞**。`buy_cat` 和 `buy_cat_entry` 这两个函数都通过调用 `create` 函数间接使用了 `aptos_framework::randomness` API，并且它们作为公共接口（`public fun` 或 `entry fun`）没有添加 `#[lint::allow_unsafe_randomness]` 属性来明确表示接受 test-and-abort 的风险。这使得恶意用户有可能通过重复尝试并中止交易来操纵最终获得的 NFT 图片。

### 项目7 harshbhatt18_FA_aptos

```
 entry fun airdrop_tokens_with_randomness(
        admin: &signer, amount: u64
    ) acquires ManagedFungibleAsset, Features {
        let features = borrow_global<Features>(object::object_address(&get_metadata()));
        let no_of_whitelisted_addresses =
            smart_vector::length(&features.whitelisted_addresses_keys);
        let random_index = randomness::u64_range(0, no_of_whitelisted_addresses);
        let random_address =
            *smart_vector::borrow(&features.whitelisted_addresses_keys, random_index);
        let to_vector = vector::singleton(random_address);
        let amount_vector = vector::singleton(amount);
        airdropTokens(admin, to_vector, amount_vector);
    }
```

**存在漏洞**。这是一个典型的 test-and-abort 场景。调用此函数的 `admin` 可以在交易确认前，通过模拟执行或其他手段预测 `random_index` 的值，从而得知哪个地址 (`random_address`) 将被选中接收空投。如果选中的地址不是 `admin` 想要的（例如，`admin` 可能希望空投给某个特定的地址，或者避免某个地址），`admin` 可以中止交易并重新发起调用，直到 `randomness::u64_range` 返回一个导致选中期望地址的 `random_index`。这完全破坏了随机空投的公平性和不可预测性。

### 项目8 hien17_slot-game-aptos-move

```
 #[randomness]
    entry fun make_random_slot_commit(game_id: u64) acquires Game, RandomnessCommitmentExt {
        assert_game_exist(game_id);
        let game_address = get_game_address(game_id);

        let exist_randomness_commitment_ext = exists<RandomnessCommitmentExt>(game_address);

        let (random_slot_1_value, random_slot_2_value, random_slot_3_value) = get_random();

        if (exist_randomness_commitment_ext) {
            let random_commitment_ext = borrow_global_mut<RandomnessCommitmentExt>(game_address);
            // Randomness should already be revealed now so it can be committed again
            // Throw error if it's already committed but not revealed
            assert!(random_commitment_ext.revealed, EALREADY_COMMITTED);
            // Commit a new random value now, flip the revealed flag to false
            random_commitment_ext.revealed = false;
            vector::push_back(&mut random_commitment_ext.values, random_slot_1_value);
            vector::push_back(&mut random_commitment_ext.values, random_slot_2_value);
            vector::push_back(&mut random_commitment_ext.values, random_slot_3_value);
        }
        else {
            let values = vector::empty<u8>();
            vector::push_back(&mut values, random_slot_1_value);
            vector::push_back(&mut values, random_slot_2_value);
            vector::push_back(&mut values, random_slot_3_value);
            let game_signer_ref = get_game_signer(game_address);
            move_to(&game_signer_ref, RandomnessCommitmentExt{
                revealed: false,
                values: values,
            });
        }
    }

    fun get_random(): (u8, u8, u8) {
        let random_slot_1_value = randomness::u8_range(0, 31);
        let random_slot_2_value = randomness::u8_range(0, 31);
        let random_slot_3_value = randomness::u8_range(0, 31);
        (random_slot_1_value, random_slot_2_value, random_slot_3_value)
    }
```

它通过调用 `get_random` **直接**使用了 `randomness::u8_range` 来生成 slot 值。

**漏洞确认:** **存在漏洞，且被审计报告遗漏！** 这是典型的 test-and-abort 漏洞。用户调用此函数时，随机数在 commit 阶段就被生成了。用户可以在交易最终确认前预测或观察到生成的随机值（存储在 `RandomnessCommitmentExt` 中），如果不满意这个即将被“承诺”的结果，可以中止交易并重试，直到获得一组有利的 slot 值再让 commit 成功。这使得 Commit-Reveal 方案的安全性被破坏。

### 项目9 imacpowers_APTOS-RANDOMNESS

```
    #[randomness]
    entry fun randomly_set_computer_move(account: &signer) acquires Game {
        randomly_set_computer_move_internal(account);
    }

```

**存在漏洞**。是一个典型的 test-and-abort 场景

### 项目10 jasonhedman_aptosino

```
    entry fun roll_dice(
        player: &signer, 
        bet_amount_input: u64,
        max_outcome: u64,
        predicted_outcome: u64
    ) {
        let result = randomness::u64_range(0, max_outcome);
        roll_dice_impl(player, bet_amount_input, max_outcome, predicted_outcome, result);
    }
    
    
        /// Spins the wheel and pays out the player
    /// * player: the signer of the player account
    /// * bet_amount_inputs: the amount to bet on each predicted outcome
    /// * predicted_outcomes: the numbers the player predicts for each corresponding bet
    entry fun spin_wheel(
        player: &signer,
        bet_amount_inputs: vector<u64>,
        predicted_outcomes: vector<vector<u8>>,
    ) {
        let result = randomness::u8_range(0, NUM_OUTCOMES);
        spin_wheel_impl(player, bet_amount_inputs, predicted_outcomes, result);
    }
    
    
        /// Creates and verifies the mines board and pays out the player accordingly
    /// * player: the signer of the player account
    /// * mines_board: the mines board
    /// * predicted_outcomes: the coordinates of the cells to select
    /// * bet_amount: the amount to bet
    entry fun select_cell(
        player: &signer,
        predicted_row: u8,
        predicted_col: u8,
    ) 
    acquires MinesBoard {
        let player_address = signer::address_of(player);
        let mines_board_obj = get_mines_board_object(player_address);
        let mines_board = borrow_global<MinesBoard>(object::object_address(&mines_board_obj));
        assert_predicted_outcome_is_valid(predicted_row, predicted_col, mines_board);
        let is_mine = randomness::u8_range(0, remaining_cells(mines_board)) < mines_board.num_mines;
        select_cell_impl(player_address, predicted_row, predicted_col, is_mine);
    }
```

**存在漏洞**。该项目在 `dice`, `roulette`, `mines` 模块的核心游戏入口函数 (`roll_dice`, `spin_wheel`, `select_cell`) 中**存在明确的、可利用的随机性滥用漏洞**。

### 项目11  jedusor1_carnethealthcare-aptos

```
 // Function to create the CarnetRxPrescription collection
    fun create_carnet_rx_prescription_collection(creator: &signer) {
        let description = utf8(CARNET_RX_COLLECTION_DESCRIPTION);
        let name = utf8(CARNET_RX_COLLECTION_NAME);
        let uri = utf8(CARNET_RX_COLLECTION_URI);

        collection::create_unlimited_collection(
            creator,
            description,
            name,
            option::none(),
            uri,
        );
    }


```

**存在漏洞**。根据 Aptos 的安全准则和 test-and-abort 漏洞的定义，这个 `entry fun` 函数间接调用了链上随机性 API (`randomness::random_string`) 却没有使用 `#[lint::allow_unsafe_randomness]` 属性明确标示风险，因此构成了漏洞。

### 项目12 kinglionX69_jungle_gem_CC

```
  #[lint::allow_unsafe_randomness]
    #[randomness]
    entry fun burn_chest(
        user: &signer,
        paw_meter: u64,
        token: Object<ChestToken>
    ) acquires ChestToken, ContractData, ManagedFungibleAsset {
		。。。
        let chest_reward = aptos_framework::randomness::u64_range(
            chest_data.start_range,
            chest_data.end_range
        );
     	 。。。
    }
```

**不存在漏洞 (根据定义):** 虽然 `burn_chest` 是一个 `entry fun` 并且直接调用了随机性 API，但它**包含了 `#[lint::allow_unsafe_randomness]` 属性**。

### 项目13 kinglionX69_JungleGems-Dora

同项目13

### 项目14 Kizito2001_Aptos-bounty

```
    #[randomness]
    entry fun randomly_set_computer_move(account: &signer) acquires Game {
        randomly_set_computer_move_internal(account);
    }
```

**存在漏洞**。该函数作为 `entry fun` 间接调用了 `randomness::u8_range`，但缺少 `#[lint::allow_unsafe_randomness]` 属性，使得用户可以通过中止和重试交易来操纵计算机的出拳结果。

### 项目15  movebit_aptos_move_dataset

同项目3

### 项目16 MyDreamStarter_Dreamstarter_Aptos

```
/// Create a generator. Can be used to derive up to MAX_U16 * 32 random bytes.
    public fun new_generator(r: &Random, ctx: &mut TxContext): RandomGenerator {
        let inner = load_inner(r);
        let seed = hmac_sha3_256(
            &inner.random_bytes,
            &ctx.fresh_object_address().to_bytes()
        );
        RandomGenerator { seed, counter: 0, buffer: vector[] }
    }
```

**误报**，没有调用 Aptos 框架的随机性 API

### 项目17 nga984_aptos-blackjack

同项目1

### 项目18 panoptisDev_punktaro-app-aptos

同项目16

### 项目19  pmahanty-1983_aptos-examples

```
 /// supra vrf calls this function
    public entry fun pick_winner(
        nonce: u64,
        message: vector<u8>,
        signature: vector<u8>,
        caller_address: address,
        rng_count: u8,
        client_seed: u64,
    ) acquires Lottery {

        let lottery_resource_address = get_resource_address();
        assert!(exists<Lottery>(lottery_resource_address), error::not_found(LOTTERY_NOT_EXIST));

        let verified_num = supra_vrf::verify_callback(nonce, message, signature, caller_address, rng_count, client_seed);

        let lottery = borrow_global_mut<Lottery>(lottery_resource_address);
        lottery.random_number = option::some(verified_num);

        let random_number = *vector::borrow(&verified_num, 0);
        let random_index = random_number % vector::length(&lottery.participants);
        let winner_address = *vector::borrow(&lottery.participants, random_index);
        lottery.winner = option::some(winner_address);

        let resource_signer = account::create_signer_with_capability(&lottery.signer_cap);
        aptos_account::transfer(&resource_signer, winner_address, lottery.amount);
    }
```

**误报**，审计工具可能错误地将处理随机数（即使是来自外部 VRF）的 `entry` 函数都标记为有风险，或者未能区分 `aptos_framework::randomness` 和外部随机性源（如 `supra_vrf`）。

### 项目20 ranasadam_aptos-assets-v2

同项目12

### 项目21 Rushikeshnimkar_Aptos-dreamstarter

同项目16

### 项目22 Sahilgill24_AptoFL

```
    // Add encrypted value and randomness
    public entry fun add_encrypted_value(account: &signer, encrypted_value: vector<u8>, r: vector<u8>) acquires EncryptedData {
        let encrypted_data = borrow_global_mut<EncryptedData>(std::signer::address_of(account));
        vector::push_back(&mut encrypted_data.encrypted_values, encrypted_value);
        vector::push_back(&mut encrypted_data.randomness, r);
    }
```

**误报**。 `add_encrypted_value` 函数没有调用 `aptos_framework::randomness` API

### 项目23 Shachindra_VirtueGaming

```
public entry fun roll_dice(user: &signer, game_id: u64, avatar_obj_add: address) acquires State, Avatar{
        assert_game_initialized(game_id);
        let state = borrow_global_mut<State>(@deployer);
        let game = simple_map::borrow_mut(&mut state.games, &game_id);
        assert_game_has_started(game.is_started);
        assert_game_not_finished(game.is_finished);
        assert_avatar_owner(signer::address_of(user), object::owner(object::address_to_object<Avatar>(avatar_obj_add)));
        let avatar = borrow_global_mut<Avatar>(avatar_obj_add);
        assert_passed_interval_timeframe(avatar.last_roll_timestamp, game.customs.interval);

        let rolled = get_new_number();
        avatar.last_rolled_num = rolled;

        event::emit_event<RollDiceEvent>(
            &mut game.roll_dice_events,
            RollDiceEvent{
                game_id,
                user: signer::address_of(user),
                number: rolled,
                timestamp: timestamp::now_seconds()
            }
        );

        avatar.last_roll_timestamp = timestamp::now_seconds();
    }

```

**存在漏洞**。`public entry fun` 通过调用 `get_new_number` 间接使用了 `randomness::u64_range` API

### 项目24 supervlabs_trusted-loot-box

```
    public fun roll_u64(min_incl: u64, max_excl: u64): (u64, RandOutput) {
        let num = randomness::u64_range(min_incl, max_excl);
        let output = RandOutput {
            output: num, min: min_incl, max: max_excl,
        };
        (num, output)
    }
```

**存在漏洞**。 虽然它不是一个 `entry` 函数，不能被用户直接利用，但作为一个 `public` 函数，它直接调用了 `randomness::u64_range` 却没有添加 `#[lint::allow_unsafe_randomness]` 属性

### 项目25 sweetim_count-fi

```
    public entry fun increment(user: &signer, collection_id: u32) acquires CountCollection {
        perform_action(user, collection_id, COUNT_ACTION_INCREMENT);
    }
    
    public entry fun decrement(user: &signer, collection_id: u32) acquires CountCollection {
        perform_action(user, collection_id, COUNT_ACTION_DECREMENT);
    }
    
    #[randomness]
    entry fun random(user: &signer, collection_id: u32) acquires CountCollection {
        perform_action(user, collection_id, COUNT_ACTION_RANDOM);
    }
    

```

**误报**，代码实际上并未使用 Aptos 框架的随机性 API，而是使用了基于时间戳的伪随机性

### 项目26 therealbryanho_move-aptos-legends

```
 // Entry function for the battle
    #[randomness]
    entry fun battle(
        player: &signer,
        ap: u8,
        dp: u8,
        npcap: u8,
        npcdp: u8
    ) acquires ModuleData {
        let module_data = borrow_global_mut<ModuleData>(@mint_nft);
        let player_address = signer::address_of(player);

        // Roll for the player and NPC using randomness
        let player_roll = aptos_framework::randomness::u8_range(0, 10);
        let npc_roll = aptos_framework::randomness::u8_range(0, 10);

        // Calculate the battle values, ensuring they are not negative
        let player_value = player_roll + ap - dp;
        let npc_value = npc_roll + npcap - npcdp;

        // Determine the winner based on the calculated values
        let winner_address = if (player_value > npc_value) {
            player_address
        } else {
            @0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
        };

        // Emit an event with the winner's address
        // let event_handle = &mut module_data.token_minting_events;
        // event::emit(event_handle, BattleEvent { winner: winner_address });
        event::emit(BattleEvent { winner: winner_address });

    }
    


   /// Mint an NFT to the receiver.
    /// `mint_proof_signature` should be the `MintProofChallenge` signed by the admin's private key
    /// `public_key_bytes` should be the public key of the admin
    #[randomness]
    entry fun mint_nft(receiver: &signer, quantity: u64) acquires ModuleData {
  				...

                let attack_points = aptos_framework::randomness::u8_range(0, 10);
                let defence_points = aptos_framework::randomness::u8_range(0, 10);

                ...
            
    }

```

**存在漏洞**。这两个 `entry` 函数都直接调用了 `aptos_framework::randomness::u8_range`，但都没有添加 `#[lint::allow_unsafe_randomness]` 属性，使得用户可以通过 test-and-abort 操纵随机结果（战斗胜负或 NFT 属性）。

### 项目27 tinydom50_aptosrockpaperscissors

同项目2

### 项目28 ugendar07_Aptos-RandHack

```
#[randomness]
      entry fun royale_crash(
        player: &signer,
        stake: u64, 
        target: u64, 
        side: bool,
      ) acquires  Casinobook {
        let accountKey = MarketAccountKey {
            protocolAddress: @RandHack,
            userAddress: address_of(player),
        };
        // todo a assert test enough funds both sides :)
        assert!(enough_funds_account(player, &accountKey,stake), ERR_CUSTOMER_NOT_ENOUGH_FUNDS);
        crash(player, accountKey,stake, target,side)
//   let myroll= fixed_point64_with_sign::create_from_rational( fixed_point64_with_sign::get_raw_value(new_roll) , 1000000000000000000000, fixed_point64_with_sign::is_positive(new_roll));


    }

```

**存在漏洞**，该函数作为 `entry fun` 通过调用 `crash` 间接使用了 `randomness::u64_range` API，但缺少 `#[lint::allow_unsafe_randomness]` 属性。这使得玩家可以通过中止和重试交易来操纵 Crash 游戏中的随机结果 `price`。

### 项目29 Vinscavon_aptos_randomness_api

同项目2

### 项目30 Zona-Tres_meme-museum-aptos

```
    //Funciones externas
    #[randomness]
    entry fun crear_meme(
        firma: &signer, 
        titulo: String, 
        url: String,
    ) acquires Museo {
        inicializar_museo(firma);

        let id: u16 = obtener_id_random();
        let creado_por: address = signer::address_of(firma);
        let memes = borrow_global_mut<Museo>(creado_por);

        simple_map::add(&mut memes.memes, id, Meme {
            id,
            creado_por,
            titulo,
            url,
        });
    }

    #[view]
    public fun obtener_memes(direccion: address): vector<Meme> acquires Museo {
        let museo_personal = borrow_global<Museo>(direccion);
        simple_map::values(&museo_personal.memes)
    }

```

**存在漏洞**。根据 Aptos 的安全准则和 test-and-abort 漏洞的定义，这个 `entry fun` 函数间接调用了链上随机性 API (`randomness::u16_integer`) 却没有使用 `#[lint::allow_unsafe_randomness]` 属性明确标示风险，因此构成了漏洞。

