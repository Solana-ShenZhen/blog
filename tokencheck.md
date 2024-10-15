# 教你代币安全监测

炒币的时候经常会遇到一些盘，如果你盲目的冲进去，可能就完蛋了。比如说 `https://dexscreener.com/solana/36kqsauU1HLRxecizvXWMosV1t1mQEeaeXDEfZLvkRY4` 这个盘，最高的时候涨到773.2M，然后迅速砸盘，一切都发生在三个小时以内。这就是因为他的mint authority还存在，每当有人买入，他就迅速冻结那个人币，导致别人只能买不能卖。你现在看这个盘可能发现mint authority已经是none了，但是之前是还有的，他修改authority是`https://solscan.io/tx/3v3EuUm19Vii9vNNfexvgg7rFgy1wcmtGQemRZ94yUsk1awY9eLRGPpVUPMPxmL5WsDUi5Du8a5Ray4V1eCb55ot`这个交易。

本文将简单判断代币是否是貔貅盘以及计算计算 LP 燃烧的比例。判断代币是否还有mint权限是看`mint authority` ,判断代币是否是貔貅盘，本质上是看代币的  `freeze authority`是否还在。计算LP燃烧的比例则是直接通过pool的address，来看这个池子里的lp供应量（supply） 和 储备量，来计算整个的比例判断burn了的比例
先引入一些包

```ts
import { nu64, struct, u32, u8, Layout } from "@solana/buffer-layout";
import { Connection, PublicKey } from "@solana/web3.js";
import type { ParsedAccountData } from "@solana/web3.js";
import { LIQUIDITY_STATE_LAYOUT_V4 } from "@raydium-io/raydium-sdk";
import dotenv from "dotenv";
import { logger } from "./logger";
```

然后自定义boollayout 以及 publickeylayout类， 这两个的作用主要是定义在buffer中如何读取和存储这两个类型的数据

```ts
class PublicKeyLayout extends Layout<PublicKey> {
  constructor(property: string) {
    super(32, property);
  }

  decode(buffer: Buffer, offset = 0): PublicKey {
    return new PublicKey(buffer.slice(offset, offset + this.span));
  }

  encode(src: PublicKey, buffer: Buffer, offset = 0): number {
    buffer.set(src.toBytes(), offset);
    return this.span;
  }
}

const publicKey = (property: string) => new PublicKeyLayout(property);

class BoolLayout extends Layout<boolean> {
  constructor(property: string) {
    super(1, property);
  }

  decode(buffer: Buffer, offset = 0): boolean {
    return buffer[offset] === 1;
  }

  encode(src: boolean, buffer: Buffer, offset = 0): number {
    buffer[offset] = src ? 1 : 0;
    return this.span;
  }
}

const bool = (property: string) => new BoolLayout(property);
```
接下来就是定义token的结构
```ts
export interface Token {
  mintAuthorityOption: 1 | 0;
  mintAuthority: PublicKey;
  supply: bigint;
  decimals: number;
  isInitialized: boolean;
  freezeAuthorityOption: 1 | 0;
  freezeAuthority: PublicKey;
}

export const MintLayout = struct<Token>([
  u32("mintAuthorityOption"),
  publicKey("mintAuthority"),
  nu64("supply"),
  u8("decimals"),
  bool("isInitialized"),
  u32("freezeAuthorityOption"),
  publicKey("freezeAuthority"),
]);
```

解析直接调用decode方法即可

```ts
export const fetchAndParseMint = async (
  mint: PublicKey,
  solanaConnection: Connection
): Promise<Token | null> => {
  try {
    let { data } = (await solanaConnection.getAccountInfo(mint)) || {};
    if (!data) return null;

    return MintLayout.decode(data);
  } catch {
    return null;
  }
};
```
获取池子的信息 然后计算比例
```ts
export const fetchLiqudityPoolState = async (
  pool: PublicKey,
  solanaConnection: Connection
) => {
  try {
    const acc = await solanaConnection.getMultipleAccountsInfo([
      new PublicKey(pool),
    ]);
    const parsed = acc.map((v: any) =>
      LIQUIDITY_STATE_LAYOUT_V4.decode(v.data)
    );

    const lpMint = parsed[0].lpMint;
    const lpReserve = parsed[0].lpReserve;

    const accInfo = await solanaConnection.getParsedAccountInfo(
      new PublicKey(lpMint)
    );
    const mintInfo = (accInfo.value?.data as ParsedAccountData)?.parsed?.info;

    const lpReserve2 = lpReserve / Math.pow(10, mintInfo?.decimals);
    const actualSupply = mintInfo?.supply / Math.pow(10, mintInfo?.decimals);

    //Calculate burn percentage
    const maxLpSupply = Math.max(actualSupply, lpReserve - 1);
    const burnAmt = lpReserve - actualSupply;
    const burnPct = (burnAmt / lpReserve) * 100;

    return burnPct;
  } catch {
    return null;
  }
};
```
完整的代码可以参考[我的github](`https://github.com/brooke007/tokencheck`)