# -HD-Video-download-

video-downloader/
├── package.json
├── server.js
└── public/
    └── index.html

    mkdir video-downloader
cd video-downloader
npm init -y
npm install express cors yt-dlp-exec


const express = require('express');
const cors = require('cors');
const ytDlp = require('yt-dlp-exec');
const path = require('path');

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(cors());
app.use(express.json());
app.use(express.static('public')); // Frontend static files serve karne ke liye

// 1. Video ki details/info nikalne ke liye API
app.post('/api/info', async (req, res) => {
    const { url } = req.body;

    if (!url) {
        return res.status(400).json({ error: 'Video URL zaroori hai.' });
    }

    try {
        // Video info extract karna (download kiye bina)
        const output = await ytDlp(url, {
            dumpSingleJson: true,
            noCheckCertificates: true,
            noWarnings: true,
            preferFreeFormats: true,
            addHeader: ['referer:youtube.com', 'user-agent:googlebot']
        });

        // Useful metadata format karke bhej rahe hain
        res.json({
            title: output.title,
            thumbnail: output.thumbnail,
            duration: output.duration_string,
            uploader: output.uploader,
            formats: output.formats
                .filter(f => f.vcodec !== 'none' && f.acodec !== 'none') // Video + Audio formats
                .map(f => ({
                    format_id: f.format_id,
                    ext: f.ext,
                    resolution: f.resolution || `${f.width}x${f.height}`,
                    filesize: f.filesize ? (f.filesize / (1024 * 1024)).toFixed(2) + ' MB' : 'N/A',
                    url: f.url
                }))
        });
    } catch (error) {
        console.error('Error fetching video info:', error);
        res.status(500).json({ error: 'Video detail nikalne me dikkat aayi. Link check karein.' });
    }
});

// 2. Direct Video Download/Stream karne ke liye API
app.get('/api/download', async (req, res) => {
    const videoUrl = req.query.url;
    const formatId = req.query.format || 'best';

    if (!videoUrl) {
        return res.status(400).send('Video URL zaroori hai.');
    }

    try {
        res.header('Content-Disposition', 'attachment; filename="video.mp4"');
        
        // Direct stream ko user ko pipe karna
        const subprocess = ytDlp.exec(videoUrl, {
            format: formatId,
            output: '-'
        });

        subprocess.stdout.pipe(res);
    } catch (error) {
        console.error('Download error:', error);
        res.status(500).send('Download me dikkat aayi.');
    }
});

app.listen(PORT, () => {
    console.log(`Server chalu ho gaya hai: http://localhost:${PORT}`);
});
node server.js
