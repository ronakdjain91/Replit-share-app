<script>
  import { onMount } from 'svelte';
  import QRCode from 'qrcode';
  import { Html5Qrcode } from 'html5-qrcode';
  import { fade, fly } from 'svelte/transition';

  let peer;
  let peerId;
  let file;
  let qrCodeUrl;
  let conn;
  let receivedFileChunks = [];
  let receivedFileMetadata = null;
  let receivedSize = 0;
  let downloadUrl = null;
  let progress = 0;
  let status = 'Initializing...';
  let html5QrCode;
  let uiState = 'initial'; // initial, sending, receiving
  let theme = 'light';

  function toggleTheme() {
      theme = theme === 'light' ? 'dark' : 'light';
      localStorage.setItem('theme', theme);
      if (theme === 'dark') {
          document.documentElement.classList.add('dark');
      } else {
          document.documentElement.classList.remove('dark');
      }
      // Re-generate QR code with new theme colors
      if (peerId) generateQRCode(peerId);
  }

  function formatBytes(bytes, decimals = 2) {
    if (bytes === 0) return '0 Bytes';
    const k = 1024;
    const dm = decimals < 0 ? 0 : decimals;
    const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB'];
    const i = Math.floor(Math.log(bytes) / Math.log(k));
    return parseFloat((bytes / Math.pow(k, i)).toFixed(dm)) + ' ' + sizes[i];
  }

  onMount(async () => {
    if (localStorage.getItem('theme') === 'dark' || (!('theme' in localStorage) && window.matchMedia('(prefers-color-scheme: dark)').matches)) {
        theme = 'dark';
        document.documentElement.classList.add('dark');
    } else {
        theme = 'light';
        document.documentElement.classList.remove('dark');
    }

    const { Peer } = await import('peerjs');
    peer = new Peer();

    peer.on('open', (id) => {
      peerId = id;
      status = 'Ready to connect';
      generateQRCode(id);
    });

    peer.on('connection', (newConnection) => {
      conn = newConnection;
      status = 'Peer connected!';

      // If we are in the sending state, this is the receiver we've been waiting for.
      if (uiState === 'sending' && file) {
          sendFile();
      }
    });
  });

  function generateQRCode(id) {
      const qrCodeColor = theme === 'dark' ? '#FFFFFF' : '#020617';
      const qrCodeBg = theme === 'dark' ? '#0f172a' : '#FFFFFF';
      QRCode.toDataURL(id, {
        errorCorrectionLevel: 'H',
        type: 'image/svg',
        width: 512,
        margin: 1,
        color: { dark: qrCodeColor, light: qrCodeBg }
      },(err, url) => {
        if (err) console.error(err);
        qrCodeUrl = url;
      });
  }

  function handleFileSelect(e) {
    file = e.target.files[0];
    if (file) {
        uiState = 'sending';
        status = `File selected: ${file.name}`;
        // If we are already connected, send the file right away
        if (conn && conn.open) {
            sendFile();
        }
    }
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
      // The data listener for the receiver is now set up here
      conn.on('data', (data) => {
        // The first message received will be the metadata
        if (data.type === 'metadata') {
            receivedFileMetadata = data.payload;
            status = `Downloading: ${receivedFileMetadata.name}`;
            progress = 0;
            downloadUrl = null;
            receivedFileChunks = [];
            receivedSize = 0;
        } else { // Subsequent messages are file chunks
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
  }

  function sendFile() {
    if (!file || !conn) return;
    status = `Sending: ${file.name}`;
    progress = 0;

    // Send metadata first
    conn.send({
        type: 'metadata',
        payload: {
            name: file.name,
            size: file.size,
            type: file.type,
        }
    });

    const fileReader = new FileReader();
    const chunkSize = 16 * 1024;
    let offset = 0;
    fileReader.onload = (event) => {
      conn.send(event.target.result);
      offset += event.target.result.byteLength;
      progress = Math.round((offset / file.size) * 100);
      if (offset < file.size) readSlice(offset);
      else status = 'File sent successfully!';
    };
    function readSlice(o) {
      const slice = file.slice(o, o + chunkSize);
      fileReader.readAsArrayBuffer(slice);
    }
    readSlice(0);
  }

  function goBack() {
      uiState = 'initial';
      if (conn) { conn.close(); conn = null; }
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

<div class="min-h-screen bg-gradient-to-br from-slate-50 to-slate-100 dark:from-slate-900 dark:to-slate-800 text-slate-800 dark:text-slate-200 flex flex-col items-center justify-center p-4 font-sans transition-colors">
  <div class="w-full max-w-lg mx-auto relative">
    <button on:click={toggleTheme} class="absolute top-0 right-0 -mt-16 p-2 rounded-full text-slate-400 hover:text-indigo-600 dark:hover:text-indigo-400 hover:bg-slate-100 dark:hover:bg-slate-800 transition-colors">
        {#if theme === 'light'}
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"></path></svg>
        {:else}
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="5"></circle><line x1="12" y1="1" x2="12" y2="3"></line><line x1="12" y1="21" x2="12" y2="23"></line><line x1="4.22" y1="4.22" x2="5.64" y2="5.64"></line><line x1="18.36" y1="18.36" x2="19.78" y2="19.78"></line><line x1="1" y1="12" x2="3" y2="12"></line><line x1="21" y1="12"x2="23" y2="12"></line><line x1="4.22" y1="19.78" x2="5.64" y2="18.36"></line><line x1="18.36" y1="5.64" x2="19.78" y2="4.22"></line></svg>
        {/if}
    </button>

    <header class="text-center mb-10" in:fly={{ y: -20, duration: 400, delay: 200 }} out:fade>
        <h1 class="text-4xl md:text-5xl font-bold text-slate-900 dark:text-white">Breeze <span class="text-indigo-600 dark:text-indigo-400 text-2xl align-middle">v2.0</span></h1>
        <p class="text-slate-500 dark:text-slate-400 mt-2 h-6">{status}</p>
    </header>

    <main class="w-full" in:fly={{ y: 20, duration: 400, delay: 300 }} out:fade>
        <div class="bg-white dark:bg-slate-800/50 p-8 rounded-2xl shadow-xl border border-slate-200 dark:border-slate-700 min-h-[450px] flex flex-col justify-center text-center">

            {#if uiState === 'initial'}
                <div class="flex flex-col md:flex-row gap-4" transition:fade>
                    <label for="file-input" class="w-full group cursor-pointer flex flex-col items-center justify-center bg-indigo-600 hover:bg-indigo-700 text-white p-8 rounded-xl transition-all">
                        <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="17 8 12 3 7 8"/><line x1="12" x2="12" y1="3" y2="15"/></svg>
                        <span class="text-xl font-semibold mt-2">Send File</span>
                    </label>
                    <input type="file" id="file-input" class="sr-only" on:change={handleFileSelect} disabled={!peerId}>

                    <button on:click={() => uiState = 'receiving'} class="w-full group flex flex-col items-center justify-center bg-slate-800 dark:bg-slate-700 hover:bg-slate-900 dark:hover:bg-slate-600 text-white p-8 rounded-xl transition-all">
                        <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" x2="12" y1="15" y2="3"/></svg>
                        <span class="text-xl font-semibold mt-2">Receive File</span>
                    </button>
                </div>
            {/if}

            {#if uiState === 'sending'}
                <div transition:fade>
                    <h2 class="text-2xl font-semibold text-slate-900 dark:text-white mb-6">Scan to Connect</h2>
                    {#if qrCodeUrl}
                        <img src={qrCodeUrl} alt="QR Code" class="w-56 h-56 mx-auto rounded-lg border-4 border-slate-200 dark:border-slate-700 p-1 bg-white" />
                    {:else}
                        <div class="w-56 h-56 bg-slate-100 dark:bg-slate-700 animate-pulse mx-auto rounded-lg"></div>
                    {/if}
                    <p class="mt-4 text-slate-500 dark:text-slate-400">Have the receiver scan this QR code.</p>
                    <div class="mt-6 h-6">
                      {#if progress > 0 && progress < 100}
                        <div class="w-full bg-slate-200 dark:bg-slate-700 rounded-full h-2">
                          <div class="bg-indigo-600 h-2 rounded-full transition-all duration-300" style="width: {progress}%"></div>
                        </div>
                      {/if}
                    </div>
                    <button on:click={goBack} class="mt-4 text-sm text-slate-500 dark:text-slate-400 hover:text-slate-800 dark:hover:text-white">Cancel</button>
                </div>
            {/if}

            {#if uiState === 'receiving'}
                <div transition:fade class="w-full">
                    {#if !conn}
                        <h2 class="text-2xl font-semibold text-slate-900 dark:text-white mb-6">Scan QR Code</h2>
                        <div id="qr-reader" class="w-full max-w-[250px] aspect-square bg-slate-100 dark:bg-slate-700 rounded-lg mx-auto"></div>
                        <button on:click={startScanner} disabled={!peerId} class="mt-6 w-full max-w-xs inline-flex items-center justify-center gap-2 bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3 px-4 rounded-lg transition-all disabled:bg-slate-400 disabled:cursor-not-allowed">
                            Start Camera
                        </button>
                        <button on:click={goBack} class="mt-4 text-sm text-slate-500 dark:text-slate-400 hover:text-slate-800 dark:hover:text-white">Back</button>
                    {:else}
                        {#if !receivedFileMetadata}
                            <div class="text-slate-500 dark:text-slate-400">
                                <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" class="mx-auto animate-pulse"><circle cx="12" cy="12" r="10"/><path d="m12 16-4-4 4-4"/><path d="M16 12H8"/></svg>
                                <p class="mt-2">Connected! Waiting for sender...</p>
                            </div>
                        {/if}
                        {#if receivedFileMetadata && !downloadUrl}
                            <div class="text-slate-500 dark:text-slate-400">
                                <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" class="mx-auto animate-pulse"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" x2="12" y1="15" y2="3"/></svg>
                                <p class="mt-2">Receiving file...</p>
                                <p class="font-bold text-lg">{receivedFileMetadata.name}</p>
                            </div>
                        {/if}
                        {#if downloadUrl}
                          <div class="text-slate-800 dark:text-slate-200">
                            <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" class="mx-auto text-green-500 mb-2"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/></svg>
                            <p class="font-bold text-lg">Download Ready</p>
                            <p class="text-sm text-slate-500 dark:text-slate-400">{receivedFileMetadata.name}</p>
                            <a href={downloadUrl} download={receivedFileMetadata.name} class="mt-6 w-full max-w-xs inline-flex items-center justify-center gap-2 bg-slate-800 dark:bg-slate-900 hover:bg-slate-700 text-white font-bold py-3 px-4 rounded-lg transition-all">
                                <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M12 17V3"/><path d="m6 11 6 6 6-6"/><path d="M19 21H5"/></svg>
                                Save to Device
                            </a>
                          </div>
                        {/if}
                        <div class="mt-6 h-6">
                            {#if progress > 0 && progress < 100}
                              <div class="w-full bg-slate-200 dark:bg-slate-700 rounded-full h-2">
                                <div class="bg-indigo-600 h-2 rounded-full" style="width: {progress}%"></div>
                              </div>
                            {/if}
                        </div>
                    {/if}
                </div>
            {/if}
        </div>
    </main>
  </div>
</div>
