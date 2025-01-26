# PearID
---

### **1. Architecture Overview**

1. **Frontend Interface (e.g., Kiosk/Web/Mobile):**
   - Built with a JavaScript framework (React/Next.js).
   - Handles wallet integration using libraries like Web3.js or Ethers.js.
   - Interfaces with the backend API and smart contracts.

2. **Backend (Rust):**
   - Handles off-chain services like document validation, facial recognition, and integration with IPFS.
   - Provides RESTful APIs or gRPC endpoints for the frontend.
   - Uses the **actix-web** or **warp** framework in Rust for API handling.

3. **Smart Contract (Solidity):**
   - Handles the NFT minting, wallet binding, and on-chain storage of metadata hashes (pointing to IPFS).
   - Interfaced using an Ethereum-compatible network.

4. **Storage Layer:**
   - **IPFS** for permanent and decentralized storage of user metadata (e.g., profile pictures, ID scans).

---

### **2. Rust Backend**

#### Dependencies (Cargo.toml)
```toml
[dependencies]
actix-web = "4.0"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1", features = ["full"] }
ipfs-api = "0.5.3"
```

#### Sample Rust Backend (API to Upload Data to IPFS)
```rust
use actix_web::{post, web, App, HttpResponse, HttpServer, Responder};
use serde::Deserialize;
use ipfs_api::IpfsClient;

#[derive(Deserialize)]
struct UploadRequest {
    file_data: String, // Base64-encoded file
    metadata: String,  // JSON string of metadata
}

#[post("/upload")]
async fn upload_to_ipfs(req: web::Json<UploadRequest>) -> impl Responder {
    let client = IpfsClient::default();
    
    // Decode base64 data
    let file_bytes = match base64::decode(&req.file_data) {
        Ok(bytes) => bytes,
        Err(_) => return HttpResponse::BadRequest().body("Invalid file data"),
    };

    // Upload to IPFS
    match client.add(file_bytes.as_slice()).await {
        Ok(response) => HttpResponse::Ok().json(response.hash),
        Err(err) => HttpResponse::InternalServerError().body(format!("IPFS error: {:?}", err)),
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| App::new().service(upload_to_ipfs))
        .bind("127.0.0.1:8080")?
        .run()
        .await
}
```

---

### **3. Solidity Smart Contract**

#### NFT Minting with Metadata Hash
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract PearID is ERC721URIStorage, Ownable {
    uint256 private _tokenIdCounter;

    mapping(address => bool) public verifiedUsers;

    event UserVerified(address indexed user, uint256 tokenId, string metadataURI);

    constructor() ERC721("PearID", "PID") {}

    function verifyAndMint(address user, string memory metadataURI) public onlyOwner {
        require(!verifiedUsers[user], "User is already verified");

        uint256 tokenId = _tokenIdCounter++;
        _safeMint(user, tokenId);
        _setTokenURI(tokenId, metadataURI);

        verifiedUsers[user] = true;
        emit UserVerified(user, tokenId, metadataURI);
    }
}
```

---

### **4. Integration**

#### End-to-End Flow:
1. **User Verification (Frontend/Backend):**
   - Upload scanned documents and facial data to the backend (Rust).
   - Store processed data on IPFS; the Rust backend returns the metadata hash.

2. **Mint NFT (Rust to Solidity):**
   - Call the `verifyAndMint` function in the Solidity contract using Rust.
   - Use libraries like `ethers-rs` for interaction.

#### Rust to Solidity Example
Add the following crate to `Cargo.toml`:
```toml
ethers = { version = "1.0", features = ["contract", "tokio-runtime"] }
```

```rust
use ethers::prelude::*;
use std::sync::Arc;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let provider = Provider::<Http>::try_from("https://your-eth-node-url")?;
    let wallet: LocalWallet = "your-private-key".parse()?;
    let client = SignerMiddleware::new(provider, wallet);
    let client = Arc::new(client);

    let contract_address: Address = "your-contract-address".parse()?;
    let abi = include_bytes!("../artifacts/PearID.json");
    let contract = Contract::new(contract_address, abi, client.clone());

    // Call verifyAndMint
    let user_address: Address = "user-wallet-address".parse()?;
    let metadata_uri = "ipfs://metadata-hash";

    let tx = contract.method::<_, ()>("verifyAndMint", (user_address, metadata_uri))?;
    let pending_tx = tx.send().await?;
    println!("Transaction hash: {:?}", pending_tx.tx_hash());

    Ok(())
}
```

---

### **5. Next Steps**
- **Testing:** Set up a testnet (e.g., Goerli, Avalanche Fuji) and deploy the Solidity contract for initial testing.
- **Kiosk Development:** Integrate the Rust backend API with the frontend kiosk interface.
- **User Education:** Develop instructional videos and support documentation for users.

This modular design ensures security, scalability, and a clear separation of responsibilities between the on-chain and off-chain components.

created with OpenAI by deusopus
