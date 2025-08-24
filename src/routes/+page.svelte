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
  let receivedSize = 0;
  let downloadUrl = null;
  let progress = 0;
  let status = 'Initializing...';
  let html5QrCode;

  onMount(async () => {
    const { Peer } = await import('peerjs');
    peer = new Peer();

    peer.on('open', (id) => {
      peerId = id;
      status = 'Ready to connect.';
      QRCode.toDataURL(id, (err, url) => {
        if (err) {
          console.error(err);
          return;
        }
        qrCodeUrl = url;
      });
    });

    peer.on('connection', (connection) => {
      conn = connection;
      status = 'Peer connected. Ready to receive file.';
      conn.on('data', (data) => {
        if (data.type === 'metadata') {
          receivedFileMetadata = data.payload;
          receivedFileChunks = [];
          receivedSize = 0;
          downloadUrl = null;
          progress = 0;
          status = `Receiving file: ${receivedFileMetadata.name}`;
        } else {
          receivedFileChunks.push(data);
          receivedSize += data.byteLength;
          progress = Math.round((receivedSize / receivedFileMetadata.size) * 100);

          if (receivedSize === receivedFileMetadata.size) {
            status = 'File received successfully!';
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
        status = `File selected: ${file.name}`;
    }
  }

  function startScanner() {
    html5QrCode = new Html5Qrcode("qr-reader");
    const qrCodeSuccessCallback = (decodedText, decodedResult) => {
        html5QrCode.stop();
        connectToPeer(decodedText);
    };
    const config = { fps: 10, qrbox: { width: 250, height: 250 } };
    html5QrCode.start({ facingMode: "environment" }, config, qrCodeSuccessCallback);
  }

  function connectToPeer(id) {
    conn = peer.connect(id);
    conn.on('open', () => {
      status = 'Connection established! Ready to send file.';
    });
  }

  function sendFile() {
    if (!file || !conn) {
      alert('Please select a file and establish a connection first.');
      return;
    }
    status = `Sending file: ${file.name}`;
    progress = 0;

    conn.send({
      type: 'metadata',
      payload: {
        name: file.name,
        size: file.size,
        type: file.type,
      },
    });

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
</script>

<div class="min-h-screen bg-gray-100 flex items-center justify-center p-4">
  <div class="max-w-4xl w-full mx-auto p-8 bg-white rounded-lg shadow-lg">
    <h1 class="text-3xl font-bold text-center text-gray-800 mb-2">Fast File Share</h1>
    <p class="text-center text-gray-500 mb-8">{status}</p>

    <div class="grid grid-cols-1 md:grid-cols-2 gap-8">
      <!-- Sender Section -->
      <div class="p-6 border border-gray-200 rounded-lg flex flex-col justify-between">
        <div>
          <h2 class="text-2xl font-semibold text-gray-700 mb-4 text-center">Send a File</h2>
          <div class="flex flex-col items-center">
            <input type="file" id="file-input" class="mb-4" on:change={handleFileSelect} disabled={!peerId}>
            {#if qrCodeUrl}
              <img src={qrCodeUrl} alt="QR Code" class="w-48 h-48" />
            {:else}
              <div class="w-48 h-48 bg-gray-200 flex items-center justify-center text-gray-500 rounded-lg">
                Generating QR Code...
              </div>
            {/if}
          </div>
        </div>
        <div class="mt-4">
          {#if progress > 0 && progress < 100}
            <div class="w-full bg-gray-200 rounded-full h-2.5">
              <div class="bg-blue-600 h-2.5 rounded-full" style="width: {progress}%"></div>
            </div>
          {/if}
          <button on:click={sendFile} disabled={!file || !conn} class="w-full mt-2 bg-green-500 hover:bg-green-600 text-white font-bold py-2 px-4 rounded-lg disabled:bg-gray-400">
            Send File
          </button>
        </div>
      </div>
      <!-- Receiver Section -->
      <div class="p-6 border border-gray-200 rounded-lg flex flex-col justify-between">
        <div>
          <h2 class="text-2xl font-semibold text-gray-700 mb-4 text-center">Receive a File</h2>
          <div class="flex flex-col items-center">
            {#if !conn}
              <div id="qr-reader" class="w-full h-48 bg-gray-200 rounded-lg mb-4"></div>
              <button on:click={startScanner} disabled={!peerId} class="bg-blue-500 hover:bg-blue-600 text-white font-bold py-2 px-4 rounded-lg disabled:bg-gray-400">
                Scan QR Code
              </button>
            {/if}
            {#if downloadUrl}
              <div class="mt-4 text-center">
                <p>File received: {receivedFileMetadata.name}</p>
                <a href={downloadUrl} download={receivedFileMetadata.name} class="text-blue-500 hover:underline">
                  Download File
                </a>
              </div>
            {/if}
          </div>
        </div>
        <div class="mt-4">
            {#if progress > 0 && progress < 100}
              <div class="w-full bg-gray-200 rounded-full h-2.5">
                <div class="bg-blue-600 h-2.5 rounded-full" style="width: {progress}%"></div>
              </div>
            {/if}
        </div>
      </div>
    </div>
  </div>
</div>
