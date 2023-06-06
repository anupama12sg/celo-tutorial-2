#  Build, Test, and Deploy a E-commerce smart contract using Hardhat on Celo Alfajores Testnet

In this tutorial, we will walk through the process of building, testing, and deploying
an E-commercesmart contract using Hardhat as the development
framework and Celo Alfajores Testnet as the blockchain network. We will use the provided smart
contract called "Mundo" for the E-commerce functionality. 

The tutorial assumes you have basic
knowledge of Ethereum development and Solidity.

Prerequisites:

- Basic knowledge of Ethereum development
- Node.js and npm installed
- Familiarity with JavaScript and Solidity

## Step 1: Set Up the Development Environment

1. Create a new directory for your project and navigate to it using the command line.
2. Initialize a new npm project by running the following command

```bash
npm init -y

```
3. Install the required dependencies by running the following commands:

```bash
npm install --save-dev hardhat @nomiclabs/hardhat-celo @nomiclabs/hardhat-web3 ethers chai @nomiclabs/hardhat-ethers dotenv
```
4. Create a new file called hardhat.config.js in the project directory and add the following configuration:

```bash
require("@nomiclabs/hardhat-waffle");
require("dotenv").config({ path: ".env" });

module.exports = {
  solidity: "0.8.4",
  networks: {
    alfajores: {
      url: "https://alfajores-forno.celo-testnet.org", 
      accounts: [process.env.PRIVATE_KEY],
      chainId: 44787,
    },
  },
  mocha: {
    timeout: 500000, // 500 seconds max for running tests
}
};
```

This configuration file sets up the networks to be used with Hardhat, including the Celo network. It retrieves the necessary configuration values from the .env file, which we will set up in the next step.

5. Create a new file named .env in the project root directory and add the following lines, replacing the placeholder values with your own:
```bash
PRIVATE_KEY=<Your_private_key>
```
Replace `<Your_private_key>` with your private key

## Step 2: Create the Smart Contract

Create a new file called `Mundo.sol` in the contracts directory. This contract will provide basic functionality for listing items, buying items, and managing orders within an E-commerce store on the blockchain.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;

contract Mundo {
    address public owner;

    struct Item {
        uint256 id;
        string name;
        string category;
        string image;
        uint256 cost;
        uint256 rating;
        uint256 stock;
        string description;
    }

    struct Order {
        uint256 time;
        Item item;
    }

    mapping(uint256 => Item) public items;
    mapping(address => mapping(uint256 => Order)) public orders;
    mapping(address => uint256) public orderCount;

    event Buy(address buyer, uint256 orderId, uint256 itemId);
    event List(string name, uint256 cost, uint256 quantity);

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function list(
        uint256 _id,
        string memory _name,
        string memory _category,
        string memory _image,
        uint256 _cost,
        uint256 _rating,
        uint256 _stock,
        string memory _description
    ) public onlyOwner {
        // Create Item
        Item memory item = Item(
            _id,
            _name,
            _category,
            _image,
            _cost,
            _rating,
            _stock,
            _description
        );

        // Add Item to mapping
        items[_id] = item;

        // Emit event
        emit List(_name, _cost, _stock);
    }

    function buy(uint256 _id) public payable {
        // Fetch item
        Item memory item = items[_id];

        // Require enough ether to buy item
        require(msg.value >= item.cost);

        // Require item is in stock
        require(item.stock > 0);

        // Create order
        Order memory order = Order(block.timestamp, item);

        // Add order for user
        orderCount[msg.sender]++; // <-- Order ID
        orders[msg.sender][orderCount[msg.sender]] = order;

        // Subtract stock
        items[_id].stock = item.stock - 1;

        // Emit event
        emit Buy(msg.sender, orderCount[msg.sender], item.id);
    }

    function withdraw() public onlyOwner {
        (bool success, ) = owner.call{value: address(this).balance}("");
        require(success);
    }

      // getters function
    function getItem(uint256 itemId)
        external
        view
        returns (Item memory)
    {
        return items[itemId];
    }
}
```

 Let's go through the main components and functionality of this contract:

1. **State Variables**

-  `owner`: Stores the address of the contract owner.

2. **Structs**:

`Item`: Represents an item in the E-commerce store. It consists of the following properties:

  - `id`: Unique identifier for the item.
  - `name`: Name of the item.
  - `category`: Category of the item.
  - `image`: URL or IPFS hash of the item's image.
  - `cost`: Cost of the item in the network's native currency.
  - `rating`: Rating of the item.
  -  `stock`: Quantity of the item available in stock.
  - `description`: Description or details of the item.
  
- `Order`: Represents an order placed by a buyer. It consists of the following properties:

   - `time`: Timestamp of when the order was created.
  - `item`: An instance of the `Item` struct representing the ordered item.
  
3. **Mappings**:

- `items`: Maps item IDs to their corresponding `Item` structs.
- `orders`: Maps the buyer's address and order ID to the corresponding `Order` struct.
- `orderCount`: Maps the buyer's address to the number of orders they have placed.

3. **Events**:

- `Buy`: Triggered when a buyer successfully purchases an item. Emits the buyer's address, order ID, and item ID.
- `List`: Triggered when a new item is listed in the store. Emits the item's name, cost, and quantity.

4. **Modifiers**:

- `onlyOwner`: A modifier that restricts the execution of a function to the contract owner.

5. **Constructor**:

Initializes the `owner` variable with the address of the contract deployer.

6. **Functions**:

- `list`: Allows the contract owner to list a new item in the store. It takes the item details as parameters and adds the item to the `items` mapping. It also emits the `List` event.
- `buy`: Allows a buyer to purchase an item. The buyer must send enough ether to cover the item's cost. It checks that the item is in stock, creates an order, and adds it to the `orders` mapping. It subtracts one from the item's stock and emits the`Buy` event.
- `withdraw`: Allows the contract owner to withdraw the contract's balance. It transfers the contract's balance to the owner's address.
- `getItem`: Retrieves the details of an item based on its ID. This function is marked as `external` and `view`, meaning it can be called from outside the contract and does not modify the contract state.

## Step 3: Write the Test File
Open the test folder in the project directory and create a new file named Mundo.test.js. Add the following code:

```javascript
const {
  time,
  loadFixture,
} = require("@nomicfoundation/hardhat-network-helpers");
const { anyValue } = require("@nomicfoundation/hardhat-chai-matchers/withArgs");
const { expect } = require("chai");
const { ethers } = require("hardhat");

const tokens = (n) => {
  return ethers.utils.parseUnits(n.toString(), 'ether')
}

// Global constants for listing an item...
const ID = 1
const NAME = "Shoes"
const CATEGORY = "Clothing"
const IMAGE = "https://www.highsnobiety.com/static-assets/thumbor/0JF865q_cG7MRaHmhMvuIdI2uz8=/1600x1067/www.highsnobiety.com/static-assets/wp-content/uploads/2021/03/16145358/main1.jpg"
const COST = tokens(1)
const RATING = 4
const STOCK = 5

describe("Mundo", function () {
  let mundo
  let deployer, buyer
  this.beforeEach(async()=>{
    // set accounts
    deployer = (await ethers.getSigners())[0]; // Retrieve owner as a signer
    buyer = '0x60031b5df905D92786dea1781E731B88b959c8A6'

    // Deploy the contract
    const Mundo = await ethers.getContractFactory('Mundo')
    mundo = await Mundo.deploy();

  })
  describe("Deployment", function () {
   it('it set the owner', async()=>{
      expect( await mundo.owner()).to.equal(deployer.address);
   })
  });
  describe('list product',()=>{
    beforeEach(async () => {
      // List a product
      transaction = await mundo.connect(deployer).list(ID, NAME, CATEGORY, IMAGE, COST, RATING, STOCK)
      await transaction.wait()
    })
    it("Returns item attributes", async () => {
      const item = await mundo.items(ID)

      expect(item.id).to.equal(ID)
      expect(item.name).to.equal(NAME)
      expect(item.category).to.equal(CATEGORY)
      expect(item.image).to.equal(IMAGE)
      expect(item.cost).to.equal(COST)
      expect(item.rating).to.equal(RATING)
      expect(item.stock).to.equal(STOCK)
    })

    it("Emits List event", () => {
      expect(transaction).to.emit(mundo, "List")
    })
  })

  describe('buys product',()=>{
    let tx;
    beforeEach(async()=>{
      // list product
      tx  = await mundo.connect(deployer).list(ID, NAME, CATEGORY, IMAGE, COST, RATING, STOCK)
      await tx.wait()

      // buy product
      const buy = await mundo.connect(buyer).buy(ID, { value: COST })
      await buy.wait()
   })

   it("Updates buyer's order count", async () => {
    const result = await mundo.orderCount(buyer.address)
    expect(result).to.equal(1)
  })

  it("Adds the order", async () => {
    const order = await mundo.orders(buyer.address, 1)

    expect(order.time).to.be.greaterThan(0)
    expect(order.item.name).to.equal(NAME)
  })

  it("Updates the contract balance", async () => {
    const result = await ethers.provider.getBalance(mundo.address)
    expect(result).to.equal(COST)
  })

  it("Emits Buy event", () => {
    expect(transaction).to.emit(mundo, "Buy")
  })

  })
  describe("Withdrawing", () => {
    let balanceBefore

    beforeEach(async () => {
      // List a item
      let transaction = await mundo.connect(deployer).list(ID, NAME, CATEGORY, IMAGE, COST, RATING, STOCK)
      await transaction.wait()

      // Buy a item
      transaction = await mundo.connect(buyer).buy(ID, { value: COST })
      await transaction.wait()

      // Get Deployer balance before
      balanceBefore = await ethers.provider.getBalance(deployer.address)

      // Withdraw
      transaction = await mundo.connect(deployer).withdraw()
      await transaction.wait()
    })

    it('Updates the owner balance', async () => {
      const balanceAfter = await ethers.provider.getBalance(deployer.address)
      expect(balanceAfter).to.be.greaterThan(balanceBefore)
    })

    it('Updates the contract balance', async () => {
      const result = await ethers.provider.getBalance(mundo.address)
      expect(result).to.equal(0)
    })
  })
})
```

Let's go through the code and understand the purpose of each section:

1. **Import Statements**:

- The necessary dependencies are imported, including `time` and `loadFixture` from `@nomicfoundation/hardhat-network-helpers`, `anyValue` from `@nomicfoundation/hardhat-chai-matchers/withArgs`, `expect` from `Chai`, and ethers from Hardhat.

2. **Helper Function**:

- The `tokens` function is defined to convert a numerical value into its equivalent representation in token units (wei).

3. **Global Constants**

- Several constants are declared to represent the details of an item, such as ID, name, category, image URL, cost, rating, and stock quantity.

4. **Test Suite**:

- The test suite is initiated using the `describe` function, encapsulating all the individual tests for the "Mundo" contract.

5. **The beforeEach Hook**:

- This hook is executed before each test case within the suite. It performs the following steps:
  - Sets the `deployer` variable to the first signer retrieved from the available signers (in this case, the owner).
  - Assigns a buyer's address to the `buyer` variable.
  - Deploys the "Mundo" contract using the `Mundo` contract factory.
6. **Deployment Test**:

- This test case verifies that the contract owner is correctly set during deployment. It checks if the owner address matches the `deployer` address.

7. **List Product** Test:

- This test case checks the functionality of listing a product. It executes the following steps:
  - Lists a product by calling the list function on the deployed contract using the deployer account.
  - Verifies that the item attributes stored in the contract match the provided values.
  - Confirms that the "List" event is emitted.
  
8. **Buy Product** Test:

- This test case focuses on buying a product. It performs the following actions:
  - Lists a product and ensures it is available for purchase.
  - Buys the product using the `buy` function, sending the required amount of ether.
  - Checks the following:
    - Updates the buyer's order count and verifies that it equals 1.
    - Adds the order to the contract and verifies the order's details.
    - Updates the contract's balance by comparing it to the cost of the item.
    - Confirms that the "Buy" event is emitted.
9. **Withdrawing** Test:

- This test case covers the withdrawal functionality. It executes the following steps:
  - Lists a product and buys it.
  - Retrieves the deployer's balance before the withdrawal.
  - Initiates a withdrawal by calling the `withdraw` function on the contract using the `deployer` account.
  - Checks the following:
    - Updates the owner's balance by comparing it to the balance before the withdrawal.
    - Verifies that the contract's balance is now zero.
These tests cover various aspects of the "Mundo" contract, including deployment, listing products, buying products, and withdrawing funds. They ensure that the contract functions as expected and provide an extra layer of confidence when working with the contract's functionality.

## Step 4: Deploy the Mundo smart contract on Celo
Open the scripts folder in the project directory and replace the code in the deploy.js file by the code below

```javascript
// We require the Hardhat Runtime Environment explicitly here. This is optional
// but useful for running the script in a standalone fashion through `node <script>`.
//
// You can also run a script with `npx hardhat run <script>`. If you do that, Hardhat
// will compile your contracts, add the Hardhat Runtime Environment's members to the
// global scope, and execute the script.
const hre = require("hardhat")
const { items } = require("../items.json")

const tokens = (n) => {
  return ethers.utils.parseUnits(n.toString(), 'ether')
}

async function main() {
  // Setup accounts
  const [deployer] = await ethers.getSigners()

  // Deploy mundo
  const Mundo = await hre.ethers.getContractFactory("Mundo")
  const mundo = await Mundo.deploy()
  await mundo.deployed()

  console.log(`Deployed Mundo Contract at: ${mundo.address}\n`)

  // Listing items...
  for (let i = 0; i < items.length; i++) {
    const transaction = await mundo.connect(deployer).list(
      items[i].id,
      items[i].name,
      items[i].category,
      items[i].image,
      tokens(items[i].price),
      items[i].rating,
      items[i].stock,
      items[i].description,
    )

    await transaction.wait()

    console.log(`Listed item ${items[i].id}: ${items[i].name}`)
  }
}

// We recommend this pattern to be able to use async/await everywhere
// and properly handle errors.
main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```
here's items.json

```json
{
  "items": [
    {
      "id": 1,
      "name": "iphone-14",
      "category": "electronics",
      "image": "https://ss7.vzw.com/is/image/VerizonWireless/iphone-14-pro-deep-purple-fall22-a?hei=400&fmt=webp",
      "price": "14",
      "rating": 4,
      "stock": 10,
      "description":"The iPhone 14 and iPhone 14 Plus feature a 6.1-inch (15 cm) and 6.7-inch (17 cm) display, improvements to the rear-facing camera, and satellite connectivity for ...A magical new way to interact with iPhone. A vital new safety feature designed to save lives. An innovative 48MP camera for mind-blowing detail."
    },
    {
      "id": 2,
      "name": "M1 macbook pro",
      "category": "electronics",
      "image": "https://pisces.bbystatic.com/image2/BestBuy_US/images/products/6450/6450853cv7d.jpg",
      "price": "2",
      "rating": 5,
      "stock": 6,
      "description":"Announced in January 2023, the MacBook Pro is brand new. The previous MacBook Pro models with chips from the M1 family were released in October 2021, but before  The Apple M1 chip redefines the 13-inch MacBook Pro. Featuring an 8-core CPU that flies through complex workflows in photography, coding, video editing"
    },
    {
      "id": 3,
      "name": "BAYC Headphone",
      "category": "electronics",
      "image": "https://i.etsystatic.com/29072056/r/il/1f7ceb/4103839229/il_fullxfull.4103839229_kje2.jpg",
      "price": "25",
      "rating": 2,
      "stock": 24,
      "description":"Atari has teamed up with sustainability-focused footwear brand Cariuma to produce a line of retro gaming sneakers. "
    },
    {
      "id": 4,
      "name": "Atari",
      "category": "clothing",
      "image": "https://www.highsnobiety.com/static-assets/thumbor/0JF865q_cG7MRaHmhMvuIdI2uz8=/1600x1067/www.highsnobiety.com/static-assets/wp-content/uploads/2021/03/16145358/main1.jpg",
      "price": "2",
      "rating": 5,
      "stock": 3,
      "description":"Get a Pair of Sneakers, Plant 2 Trees. Nurturing the planet is a cause that's very dear to us at Cariuma. That's why we decided to start our own"
    },
    {
      "id": 5,
      "name": "Sunglasses",
      "category": "clothing",
      "image": "https://ipfs.io/ipfs/QmTYEboq8raiBs7GTUg2yLXB3PMz6HuBNgNfSZBx5Msztg/sunglasses.jpg",
      "price": "10",
      "rating": 4,
      "stock": 12,
      "description":"After purchasing you will have a link to download the original file nft, nftartwork, gif, digitalart, artwork, ,photography, glasses, girl, woman, curly "
    },
    {
      "id": 6,
      "name": "Meta Jacket",
      "category": "clothing",
      "image": "https://cdn.shopify.com/s/files/1/0053/7994/8647/files/WEISMAN_JACKET_NFT_A_540x.jpg?v=1643846400",
      "price": "12",
      "rating": 4,
      "stock": 0,
      "description":"Earn royalty from Meta Jacket NFT and own a jacket IRL ... We're giving away 1 x Meta Jacket NFT to start the 2nd half of 2022! To participate "
    },
    {
      "id": 7,
      "name": "Puzzle Cube",
      "category": "toys",
      "image": "https://ipfs.io/ipfs/QmTYEboq8raiBs7GTUg2yLXB3PMz6HuBNgNfSZBx5Msztg/cube.jpg",
      "price": "5",
      "rating": 4,
      "stock": 15,
      "description":" Puzzle cubes are a wildly popular and recognizable style of brain teaser; rotational puzzles created and made famous by the Rubik's cube over 30 years ago.  "
    },
    {
      "id": 8,
      "name": "Train Set",
      "category": "toys",
      "image": "https://ipfs.io/ipfs/QmTYEboq8raiBs7GTUg2yLXB3PMz6HuBNgNfSZBx5Msztg/train.jpg",
      "price": "8",
      "rating": 4,
      "stock": 0,
      "description":"Shop our collection of trains and track sets at Mattel.com. Find all Thomas & Friends trains, track sets and more for toddlers and preschool kids!"
    },
    {
      "id": 9,
      "name": "Super Plastic",
      "category": "toys",
      "image": "https://1.bp.blogspot.com/-8o2x_nE2tc4/YC0K0BbkOeI/AAAAAAADmIU/CLZ47YHdHjMpfXFtNFCYq3NcW1jfhUK6wCPcBGAsYHg/s700/Super%2BPlastic%2BJANKY%2BCrypto%2BArt%2BNFT%2BCollection%2B01.png",
      "price": "15",
      "rating": 3,
      "stock": 12,
      "description":"@SUPERPLASTIC. World's top creator of animated celebs, vinyl toys & digital collectibles ⚡️ For a disturbingly awesome time follow Vinyl toy brand Superplastic has become well known for their Janky & Guggimon characters. Shop their limited edition collectibles here on StockX."
    }
  ]
}
```
The deploys.js script deploys the Mundo contract and lists items by calling the list function of the contract. Here's a brief explanation of the code:

1. The script imports the Hardhat Runtime Environment (`hre`) and the `items` array from the `items.json` file, which contains the details of the items to be listed.

2. The `tokens` function is defined to convert a value to its corresponding token representation using `ehers.utils.parseUnits`
3.The `main`async function is declared to handle the deployment and listing of items.
4. Inside the `main' function, the deployer's account is retrieved using  `ethers.getSigners()`
5. The `Mundo` contract factory is obtained using `hre.ethers.getContractFactory`
6. The `Mundo` contract instance is deployed using `Mundo.deploy()` and `await mundo.deployed()`.
7. The contract's address is printed to the console.
8. A loop is used to iterate over the  `items` array
9. For each item, the `lst` function of the `Mundo` ontract is called, passing the item's details as arguments.
10. The `list` function is called using `mundo.connect(deployer)`, indicating that the deployer is the caller of the function.
11. The price value is converted to tokens using the `tokens` function.
12. he transaction is waited for using `await transaction.wait()` to ensure it is mined.
13. The details of the listed item are printed to the console.
14. The script catches and handles any errors that occur during execution.
15. If an error occurs, it is logged to the console, and the process is exited with an exit code of 1.

The script essentially deploys the Mundo contract and lists the items specified in the items.json file, using the deployer's account to execute the transactions.

To deploy the contract follow the steps below:

1. our terminal, run the following command to compile the contract:
```bash
npx hardhat compile
```
2. Deploy the contract to the Celo network by running the following command:
```bash
npx hardhat run --network alfajores scripts/deploy.js
 ```
 This command executes the deployment script, which is automatically created by Hardhat in the scripts folder. 
 ![deployment output](https://i.ibb.co/K5fKpKZ/Screenshot-from-2023-06-06-17-41-47.png)
 
 3. Test the smart contract by running the command below
 
 ```bash
 npx hardhat test --network alfajores
 ```
 **Congratulations! You have successfully build , and deployed an E-commerce(`Mundo`)  smart contract with Hardhat on the Celo blockchain.**
 You can now interact with the contract using the generated artifacts and the deployed contract address.
