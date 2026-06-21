import "dotenv/config"
import { createPublicClient, http, encodeFunctionData } from "viem"
import { privateKeyToAccount } from "viem/accounts"
import * as chains from "viem/chains" // Mengimpor semua jaringan secara dinamis
import { createSmartAccountClient } from "permissionless"
import { to7702SimpleSmartAccount } from "permissionless/accounts"
import { createPimlicoClient } from "permissionless/clients/pimlico"
import { entryPoint07Address } from "viem/account-abstraction"

const PRIVATE_KEY_A   = process.env.PRIVATE_KEY
const PIMLICO_API_KEY = process.env.PIMLICO_API_KEY
const WALLET_B        = process.env.WALLET_B
const NFT_ADDRESS     = process.env.TOKEN_ADDRESS
const NFT_TOKEN_ID    = BigInt(process.env.NFT_TOKEN_ID || "1")
const SELECTED_CHAIN  = process.env.CHAIN?.toLowerCase() || "base" // Default ke base jika kosong
const SIMPLE_7702_IMPL = ""

// ABI Khusus Standar NFT ERC-721
const ERC721_ABI = [
  {
    name: "safeTransferFrom",
    type: "function",
    stateMutability: "nonpayable",
    inputs: [
      { name: "from", type: "address" },
      { name: "to", type: "address" },
      { name: "tokenId", type: "uint256" }
    ],
    outputs: []
  },
  {
    name: "ownerOf",
    type: "function",
    stateMutability: "view",
    inputs: [{ name: "tokenId", type: "uint256" }],
    outputs: [{ type: "address" }]
  },
  {
    name: "name",
    type: "function",
    stateMutability: "view",
    inputs: [],
    outputs: [{ type: "string" }]
  }
]

const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms))

async function main() {
  // 1. Deteksi dan pilih Jaringan secara dinamis
  const chain = chains[SELECTED_CHAIN]
  if (!chain) {
    throw new Error(`Jaringan '${SELECTED_CHAIN}' tidak didukung atau salah ketik di .env. Gunakan nama seperti 'base', 'optimism', atau 'arbitrum'.`)
  }

  console.log(`\n🔵 Auto-Monitor & Transfer NFT Gasless — EIP-7702 [Jaringan: ${chain.name.toUpperCase()}]\n`)

  const signerA       = privateKeyToAccount(PRIVATE_KEY_A)
  
  // URL Pimlico & RPC dibuat dinamis berdasarkan jaringan pilihan
  const pimlicoUrl    = `https://api.pimlico.io/v2/${chain.name.toLowerCase()}/rpc?apikey=${PIMLICO_API_KEY}`
  const rpcUrl        = chain.rpcUrls.default.http[0] // Mengambil RPC bawaan Viem secara otomatis

  const publicClient  = createPublicClient({ chain, transport: http(rpcUrl) })
  const pimlicoClient = createPimlicoClient({ transport: http(pimlicoUrl), entryPoint: { address: entryPoint07Address, version: "0.7" } })

  const account = await to7702SimpleSmartAccount({ client: publicClient, owner: signerA })

  console.log(`👛 EOA = Smart Account : ${account.address}`)
  console.log(`🎯 Wallet B (Tujuan)   : ${WALLET_B}\n`)

  // Membaca Nama Koleksi NFT secara on-chain
  let nftName = "NFT Collection"
  try {
    nftName = await publicClient.readContract({ address: NFT_ADDRESS, abi: ERC721_ABI, functionName: "name" })
  } catch (e) {
    console.log("⚠️ Gagal membaca nama koleksi NFT, menggunakan nama default.")
  }

  console.log(`🖼️ Koleksi NFT : ${nftName}`)
  console.log(`🆔 Target ID   : #${NFT_TOKEN_ID.toString()}`)
  console.log("⏳ Mulai monitoring kepemilikan NFT setiap 10 detik...\n")

  // Loop Monitoring Kepemilikan NFT
  while (true) {
    try {
      const currentOwner = await publicClient.readContract({ 
        address: NFT_ADDRESS, 
        abi: ERC721_ABI, 
        functionName: "ownerOf", 
        args: [NFT_TOKEN_ID] 
      })

      console.log(`[${new Date().toLocaleTimeString()}] 👤 Pemilik saat ini: ${currentOwner}`)

      // Jika pemilik NFT sudah berubah menjadi alamat Smart Account kita
      if (currentOwner.toLowerCase() === account.address.toLowerCase()) {
        console.log(`\n🚀 NFT #${NFT_TOKEN_ID.toString()} terdeteksi di wallet! Menyiapkan transfer...`)
        break 
      }
    } catch (error) {
      console.error(`❌ Belum terdeteksi / Belum di-mint: ${error.message}. Mencoba lagi...`)
    }

    await delay(10000)
  }

  // Setup client transaksi gasless
  const smartAccountClient = createSmartAccountClient({
    account, chain, bundlerTransport: http(pimlicoUrl), paymaster: pimlicoClient,
    userOperation: { estimateFeesPerGas: async () => (await pimlicoClient.getUserOperationGasPrice()).fast },
  })

  const nonce         = await publicClient.getTransactionCount({ address: signerA.address })
  const authorization = await signerA.signAuthorization({ address: SIMPLE_7702_IMPL, chainId: chain.id, nonce })

  console.log(`⏳ Mengirim NFT via safeTransferFrom gasless di jaringan ${chain.name}...`)
  
  const txHash = await smartAccountClient.sendTransaction({
    to: NFT_ADDRESS, 
    value: 0n,
    data: encodeFunctionData({ 
      abi: ERC721_ABI, 
      functionName: "safeTransferFrom", 
      args: [account.address, WALLET_B, NFT_TOKEN_ID]
    }),
    authorization,
  })

  // Link explorer otomatis menyesuaikan jaringan pilihan
  const explorerUrl = chain.blockExplorers?.default?.url || "https://etherscan.io"
  console.log(`\n✅ Berhasil Di-transfer! Tx : ${explorerUrl}/tx/${txHash}`)
  console.log(`🎉 NFT ${nftName} #${NFT_TOKEN_ID.toString()} sudah aman di Wallet B!`)
  
  process.exit(0)
}

main().catch(e => { console.error("❌ Terjadi kesalahan fatal:", e.message); process.exit(1) })
