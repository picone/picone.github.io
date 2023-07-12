---
layout: post
title: EVM 直接获取合约各变量的值
description: 本文使用 getStorageAt 来直接获取 UniswapV3 池子中各个价格区间的 liquidity。
---

# 背景

最近在研究 MEV bot，需要链下模拟 UniswapV3 的交易价格计算。问题出在 V3 Pool 的价格计算是分段的，公式中的 liquidity 也是分段的，需要知道每个
价格区间的 liquidity 才能计算出能够交易的 token 的数量。目前 GayHub 上有一个开源的模拟价格计算工具 [uniswap-v3-simulator](https://github.com/Bella-DeFinTech/uniswap-v3-simulator)，
它使用的是记录每个 mint，burn，swap 事件到 DB 然后回放从而计算出 liquidity，但是这种方法计算量较多，比较麻烦。突然看到 eth 有 getStorageAt
接口，理论上是能够获取到某个区块各个变量的值的， 从而获取到目前池子的状态。

# 步骤

1. 获取 UniswapV3Pool.sol 的 storage layout。

   ```shell
    solc --storage-layout UniswapV3Pool.sol
    ```

    输出的结果如下：   

    ```json
    {
      "storage": [
        {
          "astId": 104,
          "contract": "UniswapV3Pool.sol:UniswapV3Pool",
          "label": "slot0",
          "offset": 0,
          "slot": "0",
          "type": "t_struct(Slot0)100_storage"
        },
        {
          "astId": 108,
          "contract": "UniswapV3Pool.sol:UniswapV3Pool",
          "label": "feeGrowthGlobal0X128",
          "offset": 0,
          "slot": "1",
          "type": "t_uint256"
        },
        {
          "astId": 112,
          "contract": "UniswapV3Pool.sol:UniswapV3Pool",
          "label": "feeGrowthGlobal1X128",
          "offset": 0,
          "slot": "2",
          "type": "t_uint256"
        },
        ...
      ],
      "types": {
        "t_struct(Slot0)100_storage": {
          "encoding": "inplace",
          "label": "struct UniswapV3Pool.Slot0",
          "members": [
            {
              "astId": 87,
              "contract": "UniswapV3Pool.sol:UniswapV3Pool",
              "label": "sqrtPriceX96",
              "offset": 0,
              "slot": "0",
              "type": "t_uint160"
            },
            {
              "astId": 89,
              "contract": "UniswapV3Pool.sol:UniswapV3Pool",
              "label": "tick",
              "offset": 20,
              "slot": "0",
              "type": "t_int24"
            },
            ...
          ]
        }
      }
    }
    ```

    可以看到各个合约的 storage 级别变量存储的 slot 和类型，通过这个信息就可以获取到各个变量的值了。

2. 使用 `getStorageAt` 获取变量的值：

    我们先粗略地直接写个代码获取：

    ```python
    w3 = Web3(HTTPProvider('https://rpc-mainnet.matic.quiknode.pro'))
    for i in range(0, 9):
        res = w3.eth.get_storage_at(w3.to_checksum_address('0x4a319291e3f545089fc649f202104d406f55f27e'), i)
        print(w3.to_hex(res))
    ```
   
    输出结果如下：

    ```text
    0x000100000100010000fadc1c0000000000000000000000d021f10fae443fd581
    0x0000000000000000000000000052e78fb3e1b6f85ea88177139dce2a24edab17
    0x00000000000000000000000000000000000000b9b0738648a3e03354673372c1
    0x0000000000000000000000000000000000000000000000000000000000000000
    0x00000000000000000000000000000000000000000000000004027f7a2fa8a7c9
    0x0000000000000000000000000000000000000000000000000000000000000000
    0x0000000000000000000000000000000000000000000000000000000000000000
    0x0000000000000000000000000000000000000000000000000000000000000000
    0x010003d1be00000000f56cfd9481641e9637ae73c5ffff3905bc6c9c64ad7153
    ```

    明显地，这不是我们要的结果，针对 mapping、struct 会有别的额外操作。在 EVM 里，有一些特殊的类型：

    - mapping: mapping 不会直接存在 slot 中，而是通过 keccak256(abi.encode(key + slot)) 来计算对应存储的位置。
    - struct: struct 会被依次 abi.encode() 在 slot 中存储。默认都是存储在一个 32bytes 的块里，如果存储不完需要 slot +1。
    - array: 对应 slot 中仅存储 array 的长度，计算方法为：uint256(keccak256(abi.encode(slot))) + (index * elementSize/256)。

3. 解析 Storage 结果：

    UniswapV3 Pool 使用了一个 Tick Bitmap 来存储有流动性的 tick，然后再到 ticks 里找对应价格的 liquidity。

    ```python
    w3 = Web3(HTTPProvider('https://rpc-mainnet.matic.quiknode.pro'))

    pool_address = w3.to_checksum_address('0x4a319291e3f545089fc649f202104d406f55f27e')
    tick_spacing = 200

    # struct Slot0 {
    #     // the current price
    #     uint160 sqrtPriceX96;
    #     // the current tick
    #     int24 tick;
    #     // the most-recently updated index of the observations array
    #     uint16 observationIndex;
    #     // the current maximum number of observations that are being stored
    #     uint16 observationCardinality;
    #     // the next maximum number of observations to store, triggered in observations.write
    #     uint16 observationCardinalityNext;
    #     // the current protocol fee as a percentage of the swap fee taken on withdrawal
    #     // represented as an integer denominator (1/x)%
    #     uint8 feeProtocol;
    #     // whether the pool is locked
    #     bool unlocked;
    # }
    # Slot0 public override slot0;
    slot0 = w3.eth.get_storage_at(pool_address, 0)
    slot0_decoded = struct.unpack_from('20s3s2s2s2s1s?', bytes(reversed(slot0)), 0)  # 用 abi.decode 代替可以吗？
    slot0_item = {
        'sqrtPriceX96': int.from_bytes(slot0_decoded[0], 'little'),
        'tick': int.from_bytes(slot0_decoded[1], 'little', signed=True),
        'observationIndex': int.from_bytes(slot0_decoded[2], 'little'),
        'observationCardinality': int.from_bytes(slot0_decoded[3], 'little'),
        'observationCardinalityNext': int.from_bytes(slot0_decoded[4], 'little'),
        'feeProtocol': int.from_bytes(slot0_decoded[5], 'little'),
        'unlocked': slot0_decoded[6],
    }
    print(slot0_item)

    # uint256 public override feeGrowthGlobal0X128;
    feeGrowthGlobal0X128 = w3.to_int(w3.eth.get_storage_at(pool_address, 1))
    # uint256 public override feeGrowthGlobal1X128;
    feeGrowthGlobal1X128 = w3.to_int(w3.eth.get_storage_at(pool_address, 2))
    print('feeGrowthGlobal0X128', feeGrowthGlobal0X128, 'feeGrowthGlobal1X128', feeGrowthGlobal1X128)

    # struct ProtocolFees {
    #     uint128 token0;
    #     uint128 token1;
    # }
    # ProtocolFees public override protocolFees;
    protocolFees = w3.eth.get_storage_at(pool_address, 3)
    protocolFees_decoded = struct.unpack_from('16s16s', bytes(reversed(protocolFees)), 0)
    token0_fee = int.from_bytes(protocolFees_decoded[0], 'little')
    token1_fee = int.from_bytes(protocolFees_decoded[1], 'little')
    print('token0_fee', token0_fee, 'token1_fee', token1_fee)

    # uint128 public override liquidity;
    liquidity = w3.to_int(w3.eth.get_storage_at(pool_address, 4))
    print('liquidity', liquidity)

    tick_compressed = int(slot0_item['tick'] / tick_spacing)
    tick_word_pos = tick_compressed >> 8
    tick_bit_pos = slot0_item['tick'] & 0xFF

    # mapping(int16 => uint256) public override tickBitmap;
    tick_item = w3.eth.get_storage_at(pool_address, w3.keccak(eth_abi.abi.encode(['int16', 'uint256'], [tick_word_pos, 6])))
    word = w3.to_int(tick_item)
    # read more here:https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickBitmap.sol#L42
    cur_pos = 0
    bit_pos_list = []
    while word > 0:
        if word & 1:
            bit_pos_list.append(cur_pos)
        cur_pos += 1
        word >>= 1
    print(tick_word_pos, bit_pos_list)

    for bit_pos in bit_pos_list:
        tick_val = ((tick_word_pos << 8) + bit_pos) * tick_spacing
        # struct Info {
        #     uint128 liquidityGross;
        #     int128 liquidityNet;
        #     uint256 feeGrowthOutside0X128;
        #     uint256 feeGrowthOutside1X128;
        #     int56 tickCumulativeOutside;
        #     uint160 secondsPerLiquidityOutsideX128;
        #     uint32 secondsOutside;
        #     bool initialized;
        # }
        # mapping(int24 => Tick.Info) public override ticks;
        tick_slot = w3.to_int(w3.keccak(eth_abi.abi.encode(['int24', 'uint256'], [tick_val, 5])))
        tick_info_item = {}
        tick_storage_0 = bytes(reversed(w3.eth.get_storage_at(pool_address, tick_slot + 0)))
        tick_info_item['liquidityGross'] = int.from_bytes(tick_storage_0[0:16], 'little')
        tick_info_item['liquidityNet'] = int.from_bytes(tick_storage_0[16:], 'little', signed=True)
        tick_storage_1 = bytes(reversed(w3.eth.get_storage_at(pool_address, tick_slot + 1)))
        tick_info_item['feeGrowthOutside0X128'] = int.from_bytes(tick_storage_1, 'little')
        tick_storage_2 = bytes(reversed(w3.eth.get_storage_at(pool_address, tick_slot + 2)))
        tick_info_item['feeGrowthOutside1X128'] = int.from_bytes(tick_storage_2, 'little')
        tick_storage_3 = bytes(reversed(w3.eth.get_storage_at(pool_address, tick_slot + 3)))
        tick_info_item['tickCumulativeOutside'] = int.from_bytes(tick_storage_3[:7], 'little', signed=True)
        tick_info_item['secondsPerLiquidityOutsideX128'] = int.from_bytes(tick_storage_3[7:27], 'little')
        tick_info_item['secondsOutside'] = int.from_bytes(tick_storage_3[27:31], 'little')
        tick_info_item['initialized'] = bool(tick_storage_3[31])
        print(tick_val, tick_info_item)
    ```

    最后能够正确地输出每个价格区间的流动性，如下：

    ```text
    -338200 {'liquidityGross': 1378692111686429, 'liquidityNet': 1378692111686429, 'feeGrowthOutside0X128': 715252359749449122992324152690501403691149919, 'feeGrowthOutside1X128': 4545014169660142326410637809794, 'tickCumulativeOutside': -349498823136, 'secondsPerLiquidityOutsideX128': 1957413451444152172679873419, 'secondsOutside': 1066833, 'initialized': True}
    -337600 {'liquidityGross': 22439004494799672, 'liquidityNet': 22439004494799672, 'feeGrowthOutside0X128': 2884939903193009598449616661925247844131304, 'feeGrowthOutside1X128': 6551091952134200287602329581, 'tickCumulativeOutside': -351033070, 'secondsPerLiquidityOutsideX128': 1227182863853755082839433, 'secondsOutside': 1042, 'initialized': True}
    -331000 {'liquidityGross': 22439004494799672, 'liquidityNet': -22439004494799672, 'feeGrowthOutside0X128': 0, 'feeGrowthOutside1X128': 0, 'tickCumulativeOutside': 0, 'secondsPerLiquidityOutsideX128': 0, 'secondsOutside': 0, 'initialized': True}
    -330400 {'liquidityGross': 1815331132868769, 'liquidityNet': 1815331132868769, 'feeGrowthOutside0X128': 72023946586027394223723735577488991637380483, 'feeGrowthOutside1X128': 366852270322794899428330424889, 'tickCumulativeOutside': -48706351992, 'secondsPerLiquidityOutsideX128': 264803016948034770435391616, 'secondsOutside': 148418, 'initialized': True}
    -326800 {'liquidityGross': 8416760335593, 'liquidityNet': 8416760335593, 'feeGrowthOutside0X128': 507126785859091466968293483034462344919746080, 'feeGrowthOutside1X128': 4419238853272155884560515092794, 'tickCumulativeOutside': -218300415112, 'secondsPerLiquidityOutsideX128': 1321426321162304497353438276, 'secondsOutside': 674486, 'initialized': True}
    -323200 {'liquidityGross': 1328958239782029, 'liquidityNet': 1328958239782029, 'feeGrowthOutside0X128': 139684081309673303852196377617947586787350340, 'feeGrowthOutside1X128': 1472430653922470538929273864173, 'tickCumulativeOutside': -43786446328, 'secondsPerLiquidityOutsideX128': 249419328532144939049632635, 'secondsOutside': 136353, 'initialized': True}
    -321800 {'liquidityGross': 4234446689837540, 'liquidityNet': 4234446689837540, 'feeGrowthOutside0X128': 143804165995784089705930482859992338517384898, 'feeGrowthOutside1X128': 2424810226391873438594066817126, 'tickCumulativeOutside': -49163559027, 'secondsPerLiquidityOutsideX128': 277373568083362628479526179, 'secondsOutside': 155131, 'initialized': True}
    -320600 {'liquidityGross': 1815331132868769, 'liquidityNet': -1815331132868769, 'feeGrowthOutside0X128': 0, 'feeGrowthOutside1X128': 0, 'tickCumulativeOutside': 0, 'secondsPerLiquidityOutsideX128': 0, 'secondsOutside': 0, 'initialized': True}
    -319400 {'liquidityGross': 8416760335593, 'liquidityNet': -8416760335593, 'feeGrowthOutside0X128': 2699922488686125196066088465123129598426140, 'feeGrowthOutside1X128': 37291967874845061024304506782, 'tickCumulativeOutside': -831662384, 'secondsPerLiquidityOutsideX128': 5116858030952290782658466, 'secondsOutside': 2606, 'initialized': True}
    -317400 {'liquidityGross': 1378692111686429, 'liquidityNet': -1378692111686429, 'feeGrowthOutside0X128': 0, 'feeGrowthOutside1X128': 0, 'tickCumulativeOutside': 0, 'secondsPerLiquidityOutsideX128': 0, 'secondsOutside': 0, 'initialized': True}
    -314400 {'liquidityGross': 1328958239782029, 'liquidityNet': -1328958239782029, 'feeGrowthOutside0X128': 12908909101098013397963469932074255861397326, 'feeGrowthOutside1X128': 290532904005654289711333815495, 'tickCumulativeOutside': -4868217553, 'secondsPerLiquidityOutsideX128': 33653542180678195962656535, 'secondsOutside': 15506, 'initialized': True}
    -308000 {'liquidityGross': 4234446689837540, 'liquidityNet': -4234446689837540, 'feeGrowthOutside0X128': 0, 'feeGrowthOutside1X128': 0, 'tickCumulativeOutside': 0, 'secondsPerLiquidityOutsideX128': 0, 'secondsOutside': 0, 'initialized': True}
    ```

# References
- https://github.com/ethers-io/ethers.js/discussions/2373
