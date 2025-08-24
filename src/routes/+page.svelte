<script>
  import { onMount } from 'svelte';
  import QRCode from 'qrcode';
  import { Html5Qrcode } from 'html5-qrcode';

  let peer;
  let peerId;
  let file;
  let qrCodeUrl;
  let conn;
  let receivedFileChunks = [];
  let receivedFileMetadata = null;
  let offeredFileMetadata = null;
  let receivedSize = 0;
  let downloadUrl = null;
  let progress = 0;
  let status = 'Initializing...';
  let html5QrCode;
  let uiState = 'initial'; // initial, sending, receiving

  function formatBytes(bytes, decimals = 2) {
    if (bytes === 0) return '0 Bytes';
    const k = 1024;
    const dm = decimals < 0 ? 0 : decimals;
    const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return parseFloat((bytes / Math.pow(k, i)).toFixed(dm)) + ' ' + sizes[i];
  }

  onMount(async () => {
    const { Peer } = await import('peerjs');
    peer = new Peer();

    peer.on('open', (id) => {
      peerId = id;
      status = 'Ready to connect';
      QRCode.toDataURL(id, {
        errorCorrectionLevel: 'H',
        type: 'image/svg',
        width: 512,
        margin: 1,
        color: {
            dark:"#020617",
            light:"#FFFFFF"
        }
      },(err, url) => {
        if (err) {
          console.error(err);
          return;
        }
        qrCodeUrl = url;
      });
    });

    peer.on('connection', (connection) => {
      conn = connection;
      status = 'Peer connected';
      uiState = 'receiving'; // When someone connects to us, we are the receiver
      conn.on('data', (data) => {
        if (data.type === 'file-offer') {
            receivedFileMetadata = data.payload;
            status = `Incoming file offer`;
            progress = 0;
            downloadUrl = null;
        } else if (data.constructor === ArrayBuffer) {
            receivedFileChunks.push(data);
            receivedSize += data.byteLength;
            progress = Math.round((receivedSize / receivedFileMetadata.size) * 100);

            if (receivedSize === receivedFileMetadata.size) {
                status = 'Download complete!';
                const blob = new Blob(receivedFileChunks, { type: receivedFileMetadata.type });
                downloadUrl = URL.createObjectURL(blob);
            }
        }
      });
    });
  });

  function handleFileSelect(e) {
    file = e.target.files[0];
    if (file) {
        uiState = 'sending';
        status = `File selected: ${file.name}`;
        if (conn && conn.open) {
            offerFile();
        }
    }
  }

  function offerFile() {
    if (!file || !conn) return;
    offeredFileMetadata = {
        name: file.name,
        size: file.size,
        type: file.type,
    };
    conn.send({
        type: 'file-offer',
        payload: offeredFileMetadata
    });
    status = 'Waiting for receiver to accept...';
  }

  function requestFile() {
    if (!conn || !receivedFileMetadata) return;
    conn.send({ type: 'file-request' });
    status = `Downloading ${receivedFileMetadata.name}...`;
    receivedFileChunks = [];
    receivedSize = 0;
    progress = 0;
  }

  function startScanner() {
    if (html5QrCode?.isScanning) return;
    html5QrCode = new Html5Qrcode("qr-reader");
    const qrCodeSuccessCallback = (decodedText, decodedResult) => {
        html5QrCode.stop().catch(err => console.error("Failed to stop scanner:", err));
        connectToPeer(decodedText);
    };
    const config = { fps: 10, qrbox: { width: 250, height: 250 } };
    html5QrCode.start({ facingMode: "environment" }, config, qrCodeSuccessCallback);
  }

  function connectToPeer(id) {
    conn = peer.connect(id);
    conn.on('open', () => {
      status = 'Connection established!';
      uiState = 'sending'; // After connecting, we are in sending mode
      if (file) {
          offerFile();
      }
      conn.on('data', (data) => {
          if (data.type === 'file-request') {
              sendFile();
          }
      });
    });
  }

  function sendFile() {
    if (!file || !conn) return;
    status = `Sending: ${file.name}`;
    progress = 0;

    const fileReader = new FileReader();
    const chunkSize = 16 * 1024;
    let offset = 0;

    fileReader.onload = (event) => {
      conn.send(event.target.result);
      offset += event.target.result.byteLength;
      progress = Math.round((offset / file.size) * 100);
      if (offset < file.size) {
        readSlice(offset);
      } else {
        status = 'File sent successfully!';
      }
    };

    function readSlice(o) {
      const slice = file.slice(o, o + chunkSize);
      fileReader.readAsArrayBuffer(slice);
    }
    readSlice(0);
  }

  function goBack() {
      uiState = 'initial';
      if (conn) {
        conn.close();
        conn = null;
      }
      if (html5QrCode?.isScanning) {
          html5QrCode.stop().catch(err => console.error("Failed to stop scanner:", err));
      }
      file = null;
      progress = 0;
      status = 'Ready to connect';
      receivedFileMetadata = null;
      offeredFileMetadata = null;
      downloadUrl = null;
      receivedFileChunks = [];
      receivedSize = 0;
  }
</script>

<svelte:head>
    <title>Breeze - Fast P2P File Sharing</title>
</svelte:head>

<div class="min-h-screen bg-slate-50 text-slate-800 flex flex-col items-center justify-center p-4 font-sans">
  <div class="w-full max-w-lg mx-auto">
    <header class="text-center mb-10">
        <h1 class="text-4xl md:text-5xl font-bold text-slate-900">Breeze</h1>
        <p class="text-slate-500 mt-2 h-6">{status}</p>
    </header>

    <main class="w-full">
        <div class="bg-white p-8 rounded-2xl shadow-xl border border-slate-200 min-h-[450px] flex flex-col justify-center text-center">

            <!-- STATE: Initial -->
            {#if uiState === 'initial'}
                <div class="flex flex-col md:flex-row gap-4">
                    <label for="file-input" class="w-full group cursor-pointer flex flex-col items-center justify-center bg-indigo-600 hover:bg-indigo-700 text-white p-8 rounded-xl transition-all">
                        <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="17 8 12 3 7 8"/><line x1="12" x2="12" y1="3" y2="15"/></svg>
                        <span class="text-xl font-semibold mt-2">Send File</span>
                    </label>
                    <input type="file" id="file-input" class="sr-only" on:change={handleFileSelect} disabled={!peerId}>

                    <button on:click={() => uiState = 'receiving'} class="w-full group flex flex-col items-center justify-center bg-slate-800 hover:bg-slate-900 text-white p-8 rounded-xl transition-all">
                        <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" x2="12" y1="15" y2="3"/></svg>
                        <span class="text-xl font-semibold mt-2">Receive File</span>
                    </button>
                </div>
            {/if}

            <!-- STATE: Sending -->
            {#if uiState === 'sending'}
                <h2 class="text-2xl font-semibold text-slate-900 mb-6">Scan to Connect</h2>
                {#if qrCodeUrl}
                    <img src={qrCodeUrl} alt="QR Code" class="w-56 h-56 mx-auto rounded-lg border-4 border-slate-200 p-1" />
                {:else}
                    <div class="w-56 h-56 bg-slate-100 animate-pulse mx-auto rounded-lg"></div>
                {/if}
                <p class="mt-4 text-slate-500">Have the receiver scan this QR code.</p>
                <div class="mt-6 h-6">
                  {#if progress > 0 && progress < 100}
                    <div class="w-full bg-slate-200 rounded-full h-2">
                      <div class="bg-indigo-600 h-2 rounded-full transition-all duration-300" style="width: {progress}%"></div>
                    </div>
                  {/if}
                </div>
                <button on:click={goBack} class="mt-4 text-sm text-slate-500 hover:text-slate-800">Cancel</button>
            {/if}

            <!-- STATE: Receiving -->
            {#if uiState === 'receiving'}
                <!-- Sub-state: Scanning -->
                {#if !conn}
                    <h2 class="text-2xl font-semibold text-slate-900 mb-6">Scan QR Code</h2>
                    <div id="qr-reader" class="w-full max-w-[250px] aspect-square bg-slate-100 rounded-lg mx-auto"></div>
                    <button on:click={startScanner} disabled={!peerId} class="mt-6 w-full max-w-xs inline-flex items-center justify-center gap-2 bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3 px-4 rounded-lg transition-all disabled:bg-slate-400 disabled:cursor-not-allowed">
                        Start Camera
                    </button>
                    <button on:click={goBack} class="mt-4 text-sm text-slate-500 hover:text-slate-800">Back</button>
                {:else}
                    <!-- Sub-state: Waiting for offer -->
                    {#if !receivedFileMetadata}
                        <div class="text-slate-500">
                            <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" class="mx-auto animate-pulse"><circle cx="12" cy="12" r="10"/><path d="m12 16-4-4 4-4"/><path d="M16 12H8"/></svg>
                            <p class="mt-2">Connected! Waiting for sender...</p>
                        </div>
                    {/if}
                    <!-- Sub-state: File offered -->
                    {#if receivedFileMetadata && !downloadUrl}
                      <div class="text-slate-800">
                        <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" class="mx-auto text-slate-400 mb-2"><path d="M14.5 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7.5L14.5 2z"/><polyline points="14 2 14 8 20 8"/><line x1="16" x2="8" y1="13" y2="13"/><line x1="16" x2="8" y1="17" y2="17"/><line x1="10" x2="8" y1="9" y2="9"/></svg>
                        <p class="font-bold text-lg">{receivedFileMetadata.name}</p>
                        <p class="text-sm text-slate-500">{formatBytes(receivedFileMetadata.size)}</p>
                        <button on:click={requestFile} class="mt-6 w-full max-w-xs inline-flex items-center justify-center gap-2 bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3 px-4 rounded-lg transition-all">
                          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" x2="12" y1="15" y2="3"/></svg>
                          Accept & Download
                        </button>
                      </div>
                    {/if}
                    <!-- Sub-state: Download complete -->
                    {#if downloadUrl}
                      <div class="text-slate-800">
                        <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" class="mx-auto text-green-500 mb-2"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/></svg>
                        <p class="font-bold text-lg">Download Ready</p>
                        <p class="text-sm text-slate-500">{receivedFileMetadata.name}</p>
                        <a href={downloadUrl} download={receivedFileMetadata.name} class="mt-6 w-full max-w-xs inline-flex items-center justify-center gap-2 bg-slate-800 hover:bg-slate-900 text-white font-bold py-3 px-4 rounded-lg transition-all">
                            <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 17V3"/><path d="m6 11 6 6 6-6"/><path d="M19 21H5"/></svg>
                            Save to Device
                        </a>
                      </div>
                    {/if}
                    <div class="mt-6 h-6">
                        {#if progress > 0 && progress < 100}
                          <div class="w-full bg-slate-200 rounded-full h-2">
                            <div class="bg-indigo-600 h-2 rounded-full" style="width: {progress}%"></div>
                          </div>
                        {/if}
                    </div>
                {/if}
            {/if}
        </div>
    </main>
  </div>
</div>
