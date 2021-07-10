<template>
  <div class="hello">
    <img v-if="image" :src="image">
    <h1>{{name}} <span v-if="idx">#{{idx}}</span></h1>
    <p v-if="description" class="description">{{description}}</p>
    <br>
    <hr>
    <br>
    <p>The artwork on display can be changed by burning UBI tokens via the gallery contract.<br><br> 
    The price will increase by 24 UBI with each new display, and half every 24 hours.<br><br>
    Click below to approve burning UBI with the gallery contract, and then choose a new NFT to display for a cost of <span v-if="cost">{{cost}}</span><span v-else>...</span> UBI. <br><br>
    
    </p>

    <div v-if="connected == false">
      <button @click="connectWallet()">CONNECT WALLET</button>
    </div>
    <div id="displayWindow" v-else>

      <button v-if="approved == false" @click="approveUBI()">{{approveText}}</button>

      <div v-else>
        <label for="nft_address" >NFT Address</label>
        <input id="nft_address"  v-model="new_nft_address" type="text" placeholder="0x...">
        <br>
        <label for="nft_id">NFT Number</label>
        <input id="nft_id"  v-model="new_nft_id" type="number" placeholder="0">
        <br>
        <button @click="displayNewNFT()">{{displayText}}</button>
      </div>
    </div>
    <br>
    <br>
    <hr>
    <br><br>
    <i>UNIVERSAL BASIC GALLERY HAS BURNT {{tally/24}} DAYS OF UBI</i><br>
    <a href="https://etherscan.io/address/0x45527f75E9e69900Dd1A9C9e8af1240D4C5C9E89">CONTRACT 0x45527f75E9e69900Dd1A9C9e8af1240D4C5C9E89</a>
    <br><br><br>
    <table>
      <thead>
        <th>Block</th>
        <!-- <th>Sender</th> -->
        <th>NFT Address</th>
        <th>NFT ID</th>
      </thead>
      <tbody>
        <tr v-for="log in logs">
          <td>{{log.blockNumber}}</td>
          <!-- <td>{{log.args[0]}}</td> -->
          <td>{{log.args[1]}}</td>
          <td>{{log.args[2].toString()}}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import { ethers } from "ethers";
import axios from 'axios';

export default {
  data () {
    return {
      ubg_address: "0x45527f75E9e69900Dd1A9C9e8af1240D4C5C9E89",
      ubi_address: "0xdd1ad9a21ce722c151a836373babe42c868ce9a4",
      ubg_abi: [
          "event Displaying(address _sender, address _nft, uint256 index)",
          "function activeNFTindex() public view returns (uint256)",
          "function lastUpdate() public view returns (uint256)",
          "function cost() public view returns (uint256)",
          "function display(address _add, uint256 _idx) external",
          "function tally() public view returns (uint256)",
          "function name() public view returns (string memory)",
          "function symbol() public view returns (string memory)",
          "function tokenURI(uint256 tokenId) public view returns (string memory)",
          "function burnPaymentToken() external",
          "function updatePrice() external",
      ],
      ubi_abi : [
        "function approve(address _spender, uint256 _amount) public returns (bool)",
        "function allowance(address _sender, address _spender) public view returns (uint256)"
      ],
      ubg: false,
      name: "",
      costdeci: false,
      cost: false,
      tokenURI: false,
      image: false,
      lastUpdate: false,
      idx: false,
      description: false,
      approved: false,
      approveText: "APPROVE UBI",
      displayText: "DISPLAY NFT",
      new_nft_id: null,
      new_nft_address: '',
      prev: '',
      logs: false,
      connected: false,
      tally:0,
      provider: false,
      ubiContract: false,
      ubiWithSigner: false,
      ubgContract: false,
      ubgWithSigner: false,
    }
  },
  mounted(){
    this.setup();
  },
  methods: {
    setup(){
      console.log("setup");
      this.provider = new ethers.providers.InfuraProvider("mainnet");
      this.ubiContract = new ethers.Contract(this.ubi_address, this.ubi_abi, this.provider);
      this.ubgContract = new ethers.Contract(this.ubg_address, this.ubg_abi, this.provider);
      this.getActiveTokenLoop();
    },
    async connectWallet(){
      console.log("connecting to mm");
      window.ethereum.enable();
      this.provider = await new ethers.providers.Web3Provider(window.ethereum);
      this.signer = await this.provider.getSigner();
      // console.log(this.signer)
      // this.provider = new ethers.providers.InfuraProvider("mainnet")
      this.ubiContract = new ethers.Contract(this.ubi_address, this.ubi_abi, this.provider);
      this.ubiWithSigner = this.ubiContract.connect(this.signer);
      this.ubgContract = new ethers.Contract(this.ubg_address, this.ubg_abi, this.provider);
      this.ubgWithSigner = this.ubgContract.connect(this.signer);
      this.connected = true;
    },
    async checkUBIApproved(){
      this.signer.getAddress()
        .then((address) => this.ubiContract.allowance(address, this.ubg_address)
          .then((allowance) => {
            if(allowance.eq(this.costdeci)){
              this.approved = true;
            }
          })
        )
    },
    async approveUBI(){
      this.approveText = "APPROVING";
      this.ubiWithSigner.approve(this.ubg_address, this.costdeci)
        .then((res) =>{
          this.checkUBIApproved();
        })
    },
    async displayNewNFT(){
      this.displayText = "UPDATING";
      this.ubgWithSigner.display(this.new_nft_address, this.new_nft_id)
          .then((res) => {
            console.log(res);
          })
          .catch(err => {
            console.log(err);
          })
    },
    async getLogs(){ 
      this.logs = await this.ubgContract.queryFilter('Displaying', 12800929, 'latest')
        .then((res) =>{
          console.log(res);
          return res;
        })
    },
    async getActiveTokenLoop(){
      this.name = await this.ubgContract.name();
      if(this.name != this.prev){
        this.getLogs();
        this.idx = await this.ubgContract.activeNFTindex();
        this.tokenURI = await this.ubgContract.tokenURI(0) ;
        this.costdeci = await this.ubgContract.cost();
        this.tally = ethers.utils.formatEther(await this.ubgContract.tally());
        this.cost = ethers.utils.formatEther(this.costdeci);
        this.lastUpdate = await this.ubgContract.lastUpdate();

        if(this.connected){this.checkUBIApproved();}
        

        if(this.tokenURI.substring(0, 7) == 'ipfs://'){
          this.isIPFS = true;
          this.tokenURI = this.tokenURI.replace('ipfs://', 'https://ipfs.io/ipfs/');
        }
        console.log("loading metadata", this.tokenURI);
        var response = await axios.get(this.tokenURI)
          .then(async(res) => {
              console.log(res.data);
              if(this.isIPFS){
                var im = res.data.image.replace('ipfs://', '');
                this.image = "https://ipfs.io/ipfs/"+im;
              }else{
                this.image = res.data.image;
              }
              this.description = res.data.description;
              if(typeof res.data.name != 'undefined' && 
                 typeof res.data.description == 'undefined' ){
                this.description = res.data.name;
              }
              return res;
          })
          .catch((error)=>{
              console.log(error);
              return false;
          })
        
        this.prev = this.name;
      }else{
        console.log("no change");
      }
      
      setTimeout(this.getActiveTokenLoop, 14000);

    }
  }
}
</script>

<style scoped>
.description {
  text-align: justify;
  padding-left: 20px;
  padding-right: 20px;
}

.hello{
  text-align: center;
  margin: 0 auto;
  padding-top: 50px;
  width: 100%;
  max-width: 600px;
}

h1, h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
img {
  width: 100%;
  max-width: 400px;
  /*margin: 50px;*/
  margin-top: 80px;
  margin-bottom: 80px;
}

table {
  text-align: center;
  width: 100%;
}
tr {
  max-width: 100%;
}
td {
  max-width: 30%;
  overflow: scroll;
}
</style>
