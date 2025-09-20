# unlimitedqrscans
get now unlimited scan here
openssl req -nodes -new -x509 -keyout key.pem -out cert.pem
node server.j
unlimited-qr-scan/
├── public/
│   └── index.html
├── key.pem           # Local only
├── cert.pem          # Local only
├── server.js
├── package.json
└── Procfile          # (Heroku only)
<!DOCTYPE html>
<html>
<head>
  <title>QR Code Scanner (URL/Camera)</title>
</head>
<body>
  <h2>Scan QR from Image URL</h2>
  <form id="urlForm">
    <input type="url" name="url" placeholder="Paste image URL here" required>
    <button type="submit">Scan</button>
  </form>
  <div id="urlResult"></div>
  <h2>Scan QR from Camera</h2>
  <video id="video" width="300" height="200" autoplay></video>
  <button id="snap">Scan QR</button>
  <canvas id="canvas" hidden width="300" height="200"></canvas>
  <div id="cameraResult"></div>
  <script src="https://cdn.jsdelivr.net/npm/jsqr/dist/jsQR.js"></script>
  <script>
    // Camera QR Scan
    const video = document.getElementById('video');
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    let streaming = false;

    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
      navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } })
        .then(stream => {
          video.srcObject = stream;
          streaming = true;
        });
    }

    document.getElementById('snap').onclick = () => {
      if (!streaming) return;
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
      const code = jsQR(imageData.data, imageData.width, imageData.height);
      document.getElementById('cameraResult').textContent = code ? code.data : 'QR code not found';
    };

    // URL QR Scan (needs backend endpoint at /scan/url)
    document.getElementById('urlForm').onsubmit = async (e) => {
      e.preventDefault();
      const form = e.target;
      const url = form.url.value;
      const res = await fetch('/scan/url', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url })
      });
      const result = await res.json();
      document.getElementById('urlResult').textContent = result.data || result.error;
    };
  </script>
</body>
</html>
<!DOCTYPE html>
<html>
<head>
  <title>Unlimited QR Code Scanner</title>
</head>
<body>
  <h1>Unlimited QR Code Scanner</h1>
  <h2>Scan by Upload</h2>
  <form id="uploadForm" enctype="multipart/form-data">
    <input type="file" name="qrfile" accept="image/*" required>
    <button type="submit">Scan</button>
  </form>
  <div id="uploadResult"></div>
  <h2>Scan by Image URL</h2>
  <form id="urlForm">
    <input type="url" name="url" placeholder="Paste image URL here" required>
    <button type="submit">Scan</button>
  </form>
  <div id="urlResult"></div>
  <h2>Scan by Camera</h2>
  <video id="video" width="300" height="200" autoplay></video>
  <button id="snap">Scan QR</button>
  <canvas id="canvas" hidden width="300" height="200"></canvas>
  <div id="cameraResult"></div>
  <script src="https://cdn.jsdelivr.net/npm/jsqr/dist/jsQR.js"></script>
  <script>
    // Upload
    document.getElementById('uploadForm').onsubmit = async (e) => {
      e.preventDefault();
      const form = e.target;
      const data = new FormData(form);
      const res = await fetch('/scan/upload', { method: 'POST', body: data });
      const result = await res.json();
      document.getElementById('uploadResult').textContent = result.data || result.error;
    };

    // URL
    document.getElementById('urlForm').onsubmit = async (e) => {
      e.preventDefault();
      const form = e.target;
      const url = form.url.value;
      const res = await fetch('/scan/url', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ url })
      });
      const result = await res.json();
      document.getElementById('urlResult').textContent = result.data || result.error;
    };

    // Camera
    const video = document.getElementById('video');
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    let streaming = false;

    if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
      navigator.mediaDevices.getUserMedia({ video: { facingMode: "environment" } })
        .then(stream => {
          video.srcObject = stream;
          streaming = true;
        });
    }

    document.getElementById('snap').onclick = () => {
      if (!streaming) return;
      ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
      const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
      const code = jsQR(imageData.data, imageData.width, imageData.height);
      document.getElementById('cameraResult').textContent = code ? code.data : 'QR code not found';
    };
  </script>
</body>
</html>
{
  "name": "unlimited-qr-scan",
  "version": "1.0.0",
  "description": "HTTPS web app for unlimited QR code scanning",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "multer": "^1.4.5-lts.1",
    "qrcode-reader": "^1.0.3",
    "jimp": "^0.22.10",
    "axios": "^1.6.0",
    "https": "^1.0.0"
  }
}
const fs = require('fs');
const https = require('https');
const express = require('express');
const multer = require('multer');
const Jimp = require('jimp');
const QrCode = require('qrcode-reader');
const axios = require('axios');
const path = require('path');

const app = express();
const upload = multer({ dest: 'uploads/' });

// HTTPS credentials (Use your own SSL certs for production!)
const privateKey = fs.readFileSync('key.pem');
const certificate = fs.readFileSync('cert.pem');
const credentials = { key: privateKey, cert: certificate };

app.use(express.static('public'));
app.use(express.json());

// Scan QR from uploaded file
app.post('/scan/upload', upload.single('qrfile'), async (req, res) => {
  if (!req.file) return res.status(400).send('No file uploaded');
  try {
    const image = await Jimp.read(req.file.path);
    const qr = new QrCode();
    qr.callback = (err, value) => {
      fs.unlinkSync(req.file.path); // Clean up
      if (err || !value) return res.status(400).send({ error: 'QR code not found' });
      res.send({ data: value.result });
    };
    qr.decode(image.bitmap);
  } catch (err) {
    res.status(500).send({ error: 'Error decoding QR code' });
  }
});

// Scan QR from URL
app.post('/scan/url', async (req, res) => {
  const { url } = req.body;
  if (!url) return res.status(400).send('No URL provided');
  try {
    const imgResp = await axios({ url, responseType: 'arraybuffer' });
    const image = await Jimp.read(imgResp.data);
    const qr = new QrCode();
    qr.callback = (err, value) => {
      if (err || !value) return res.status(400).send({ error: 'QR code not found' });
      res.send({ data: value.result });
    };
    qr.decode(image.bitmap);
  } catch (err) {
    res.status(500).send({ error: 'Error decoding QR code from URL' });
  }
});

// Serve HTML interface
app.get('/', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

// Start HTTPS server
https.createServer(credentials, app).listen(443, () => {
  console.log('HTTPS QR Code Scanner running on port 443');
});
