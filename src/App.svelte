<script lang="ts">
  import { ethers } from 'ethers'
  import { SafeService, type SerializedSafeTransaction } from './lib/SafeService'
  import { onMount } from 'svelte'
  import CreateSafe from './lib/CreateSafe.svelte'
  
  let safeService = new SafeService()
  let status = ''
  let connectedAddress = ''
  let txWorker: Worker
  
  // Input fields
  let safeAddress = ''
  let recipientAddress = ''
  let transactionPayload = '0x'
  let ethValue = '0'
  let weiValue = '0'

  // Transaction data for sharing
  let serializedTxStr = ''
  let currentTx: SerializedSafeTransaction | null = null

  let activeWorkflow: 'create' | 'sign' | 'new' = 'new'

  let canSign = false

  $: {
    // Update canSign whenever currentTx or connectedAddress changes
    if (currentTx && connectedAddress) {
      canSignTransaction().then(result => {
        canSign = result
      })
    } else {
      canSign = false
    }
  }

  onMount(() => {
    if (window.ethereum) {
      window.ethereum.on('accountsChanged', handleAccountsChanged)
    }

    // Initialize web worker
    txWorker = new Worker(new URL('./lib/txWorker.ts', import.meta.url), { type: 'module' })
    txWorker.onmessage = (e) => {
      const { error, tx } = e.data
      if (error) {
        status = error
        currentTx = null
      } else {
        currentTx = tx
        if (tx) {
          status = 'Transaction data loaded'
        }
      }
    }

    // Parse URL parameters
    const urlParams = new URLSearchParams(window.location.search)
    const safeAddrParam = urlParams.get('safeAddr')
    
    if (safeAddrParam) {
      safeAddress = safeAddrParam
      handleConnect() // Auto-connect if safe address is provided
    }

    return () => {
      if (window.ethereum) {
        window.ethereum.removeListener('accountsChanged', handleAccountsChanged)
      }
      // Cleanup worker
      txWorker?.terminate()
    }
  })

  async function handleAccountsChanged(accounts: string[]) {
    if (accounts.length === 0) {
      connectedAddress = ''
      status = 'Wallet disconnected'
    } else {
      if (safeAddress) {
        try {
          await safeService.connect(safeAddress)
          connectedAddress = await safeService.getAddress()
          status = 'Account changed successfully'
        } catch (error) {
          const errorMessage = error instanceof Error ? error.message : 'Unknown error'
          status = 'Failed to reconnect: ' + errorMessage
          connectedAddress = ''
        }
      }
    }
  }

  function handleEthValueChange(event: Event) {
    try {
      const input = (event.target as HTMLInputElement).value
      if (input === '') {
        ethValue = '0'
        weiValue = '0'
        return
      }
      ethValue = input
      weiValue = ethers.utils.parseEther(input).toString()
    } catch (error) {
      // If parsing fails, keep the ETH value but set Wei to 0
      weiValue = '0'
    }
  }

  async function handleConnect() {
    try {
      status = 'Connecting...'
      
      if (!safeAddress) {
        throw new Error('Please enter a Safe address')
      }
      
      await safeService.connect(safeAddress)
      connectedAddress = await safeService.getAddress()
      status = 'Connected successfully'
    } catch (error: unknown) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error'
      status = 'Connection failed: ' + errorMessage
    }
  }

  async function handleCreateTransaction() {
    try {
      if (!recipientAddress) {
        throw new Error('Please enter a recipient address')
      }

      if (!ethers.utils.isAddress(recipientAddress)) {
        throw new Error('Invalid recipient address')
      }

      if (!ethers.BigNumber.from(weiValue).gte(0)) {
        throw new Error('Value must be a non-negative amount')
      }

      status = 'Creating transaction...'
      
      const txData = {
        to: recipientAddress,
        value: weiValue,
        data: transactionPayload.startsWith('0x') ? transactionPayload : '0x' + transactionPayload
      }
      
      const result = await safeService.createTransaction(txData)
      currentTx = result.serializedTx
      serializedTxStr = JSON.stringify(currentTx, null, 2)
      status = 'Transaction created! Share the transaction data below with other signers'
    } catch (error: unknown) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error'
      status = 'Transaction creation failed: ' + errorMessage
    }
  }

  function handleTransactionDataChange() {
    // Send data to worker for parsing
    txWorker.postMessage({ txData: serializedTxStr })
  }

  async function canSignTransaction(): Promise<boolean> {
    if (!currentTx || !connectedAddress) return false
    
    // Check if the current address hasn't signed yet
    return !currentTx.signatures.some(sig => 
      sig.signer.toLowerCase() === connectedAddress.toLowerCase()
    )
  }

  async function copyToClipboard() {
    try {
      await navigator.clipboard.writeText(serializedTxStr)
      status = 'Transaction data copied to clipboard!'
    } catch (error) {
      status = 'Failed to copy to clipboard. Please copy manually.'
    }
  }

  async function handleAddSignature() {
    try {
      if (!currentTx) {
        throw new Error('Please load a transaction first')
      }

      status = 'Adding signature...'
      const result = await safeService.addSignature(currentTx)
      currentTx = result.serializedTx
      serializedTxStr = JSON.stringify(currentTx, null, 2)
      status = 'Signature added successfully!'
    } catch (error: unknown) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error'
      status = 'Failed to add signature: ' + errorMessage
    }
  }

  async function handleExecuteTransaction() {
    try {
      if (!currentTx) {
        throw new Error('Please load a transaction first')
      }

      status = 'Executing transaction...'
      const response = await safeService.executeTransaction(currentTx)
      status = 'Transaction executed! Hash: ' + response.hash
    } catch (error: unknown) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error'
      status = 'Transaction execution failed: ' + errorMessage
    }
  }

  function getSignatureCount(tx: SerializedSafeTransaction | null): number {
    return tx?.signatures?.length || 0
  }

  function isCurrentSigner(signerAddress: string): boolean {
    return connectedAddress.toLowerCase() === signerAddress.toLowerCase()
  }
</script>

<main>
  <h1>Safe Protocol Minimal</h1>
  
  <div class="workflow-selector">
    <button 
      class="tab-button {activeWorkflow === 'new' ? 'active' : ''}"
      on:click={() => activeWorkflow = 'new'}
    >
      Create New Safe
    </button>
    <button 
      class="tab-button {activeWorkflow === 'create' ? 'active' : ''}"
      on:click={() => activeWorkflow = 'create'}
    >
      Create Transaction
    </button>
    <button 
      class="tab-button {activeWorkflow === 'sign' ? 'active' : ''}"
      on:click={() => activeWorkflow = 'sign'}
    >
      Sign Transaction
    </button>
  </div>

  {#if activeWorkflow === 'new'}
    <CreateSafe 
      {safeService} 
      on:safecreated={(e) => {
        safeAddress = e.detail.safeAddress
        activeWorkflow = 'create'
        handleConnect()
      }}
    />
  {:else}
    <div class="guide-steps">
      <div class="step">
        <span class="step-number">1</span>
        <div class="step-content">
          <h3>Connect to Safe</h3>
          <p>Enter your Safe address and connect your wallet to get started.</p>
        </div>
      </div>
    </div>

    <div class="form-group">
      <label for="safe-address">
        Safe Address:
        <span class="tooltip" data-tooltip="The address of your Safe multi-signature wallet">ⓘ</span>
      </label>
      <div class="input-group">
        <input 
          id="safe-address"
          type="text" 
          bind:value={safeAddress} 
          placeholder="Enter Safe address (0x...)"
        />
        <button class="primary-button" on:click={handleConnect}>Connect Wallet</button>
      </div>
    </div>

    {#if connectedAddress}
      {#if activeWorkflow === 'create'}
        <div class="guide-steps">
          <div class="step">
            <span class="step-number">2</span>
            <div class="step-content">
              <h3>Create Transaction</h3>
              <p>Fill in the transaction details to create a new transaction that requires signatures.</p>
            </div>
          </div>
        </div>

        <div class="section">
          <h2>Create Transaction</h2>
          <div class="form-group">
            <label for="recipient-address">
              To:
              <span class="tooltip" data-tooltip="The Ethereum address that will receive this transaction">ⓘ</span>
            </label>
            <input 
              id="recipient-address"
              type="text" 
              bind:value={recipientAddress} 
              placeholder="Recipient address (0x...)"
            />
          </div>

          <div class="form-group">
            <label for="transaction-value">
              Value (ETH):
              <span class="tooltip" data-tooltip="Amount of ETH to send">ⓘ</span>
            </label>
            <input 
              id="transaction-value"
              type="number" 
              value={ethValue}
              on:input={handleEthValueChange}
              min="0"
              step="0.000000000000000001"
              placeholder="0.0"
            />
            <span class="helper-text">Wei: {weiValue}</span>
          </div>

          <div class="form-group">
            <label for="transaction-payload">
              Data:
              <span class="tooltip" data-tooltip="Optional: Hexadecimal data for contract interactions">ⓘ</span>
            </label>
            <input 
              id="transaction-payload"
              type="text" 
              bind:value={transactionPayload} 
              placeholder="0x... (optional)"
            />
          </div>

          <button class="primary-button" on:click={handleCreateTransaction}>Create Transaction</button>
        </div>

        <div class="guide-steps">
          <div class="step">
            <span class="step-number">3</span>
            <div class="step-content">
              <h3>Share Transaction</h3>
              <p>Share the transaction data with other signers for their signatures.</p>
            </div>
          </div>
        </div>
      {:else}
        <div class="guide-steps">
          <div class="step">
            <span class="step-number">2</span>
            <div class="step-content">
              <h3>Sign Transaction</h3>
              <p>Paste the transaction data shared with you to sign it.</p>
            </div>
          </div>
        </div>
      {/if}

      <div class="section">
        <h2>Sign & Execute</h2>
        <div class="form-group">
          <label for="transaction-data">
            Transaction Data
            {#if currentTx && getSignatureCount(currentTx) > 0}
              <span class="label-badge">
                {getSignatureCount(currentTx)} signature{getSignatureCount(currentTx) > 1 ? 's' : ''}
              </span>
            {/if}
            <span class="tooltip" data-tooltip="Paste transaction data here to sign, or copy to share with other signers">ⓘ</span>
          </label>
          <div class="textarea-container">
            <textarea
              id="transaction-data"
              bind:value={serializedTxStr}
              on:input={handleTransactionDataChange}
              placeholder={activeWorkflow === 'create' 
                ? "Your transaction data will appear here after creation" 
                : "Paste transaction data here to sign it"}
              rows="8"
              class="monospace"
            />
            {#if serializedTxStr}
              <div class="button-container">
                <button class="icon-button" on:click={copyToClipboard}>
                  Copy
                </button>
              </div>
            {/if}
          </div>
          {#if currentTx && getSignatureCount(currentTx) > 0}
            <span class="helper-text">👆 Copy this data to share with other signers</span>
          {/if}
        </div>

        <div class="button-group">
          <button 
            on:click={handleAddSignature} 
            class="secondary-button"
            disabled={!currentTx || !canSign}
          >
            <span class="button-icon">✍️</span> Sign
          </button>
          <button 
            on:click={handleExecuteTransaction} 
            class="execute-button"
            disabled={!currentTx}
          >
            <span class="button-icon">🚀</span> Execute
          </button>
        </div>

        {#if currentTx}
          <div class="transaction-info">
            <h3>Current Transaction</h3>
            <div class="info-grid">
              <div class="info-row">
                <span class="info-label">To:</span>
                <code class="info-value">{currentTx.to}</code>
              </div>
              <div class="info-row">
                <span class="info-label">Value:</span>
                <code class="info-value">{ethers.utils.formatEther(currentTx.value)} ETH</code>
              </div>
              <div class="info-row">
                <span class="info-label">Data:</span>
                <code class="info-value">{currentTx.data}</code>
              </div>
              <div class="info-row">
                <span class="info-label">Signatures ({currentTx.signatures.length}):</span>
                <div class="signatures-list">
                  {#each currentTx.signatures as signature}
                    <div class="signature-item">
                      <code class={isCurrentSigner(signature.signer) ? 'current-signer' : ''}>
                        {signature.signer}
                        {#if isCurrentSigner(signature.signer)}
                          <span class="signature-badge">Your signature</span>
                        {/if}
                      </code>
                    </div>
                  {/each}
                </div>
              </div>
            </div>
            {#if !canSign && currentTx.signatures.some(sig => isCurrentSigner(sig.signer))}
              <div class="signature-status">
                <span class="status-icon">✓</span>
                You have already signed this transaction
              </div>
            {/if}
          </div>
        {/if}
      </div>
    {/if}

    {#if status || connectedAddress}
      <div class="status-bar">
        {#if status}
          <div class="status-message">
            {status}
          </div>
          {#if status.includes('Transaction created!')}
            <div class="contract-address">
              <span class="address-label">Contract:</span>
              <code class="address-value">{currentTx?.to}</code>
            </div>
          {/if}
        {/if}
        {#if connectedAddress}
          <div class="connected-address">
            <span class="address-label">Connected:</span>
            <code class="address-value">{connectedAddress}</code>
          </div>
        {/if}
      </div>
    {/if}
  {/if}
</main>
