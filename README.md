# Prerequisites:
- Web hosting with PHP 7.4+ and MySQL 5.7+
- WordPress 6.0+ installed
- SSL certificate (HTTPS)
- - WP RSS Aggregator (Feed to Post addon)
- Automatic YouTube Video Posts
- Featured Image from URL
- WordLift (for AI content enrichment)
- Auto Post Scheduler
- AdSanity (for monetization)
- <?php
/*
Plugin Name: Auto News Setup
Description: Configures automatic news aggregation
*/

// 1. Set up default categories
function auto_news_create_categories() {
    $categories = ['World', 'Politics', 'Technology', 'Business', 'Sports', 'Entertainment'];
    
    foreach ($categories as $cat) {
        if (!term_exists($cat, 'category')) {
            wp_insert_term($cat, 'category');
        }
    }
}
add_action('init', 'auto_news_create_categories');

// 2. Configure automatic featured images
function auto_news_set_featured_image($post_id) {
    if (has_post_thumbnail($post_id)) return;
    
    $post = get_post($post_id);
    $image_url = '';
    
    // Extract first image from content
    preg_match_all('/<img.+src=[\'"]([^\'"]+)[\'"].*>/i', $post->post_content, $matches);
    if (!empty($matches[1][0])) {
        $image_url = $matches[1][0];
    }
    
    // If no image, use category default
    if (empty($image_url)) {
        $categories = get_the_category($post_id);
        $default_images = [
            'World' => 'https://example.com/default-world.jpg',
            'Technology' => 'https://example.com/default-tech.jpg',
            // Add other category defaults
        ];
        
        foreach ($categories as $category) {
            if (isset($default_images[$category->name])) {
                $image_url = $default_images[$category->name];
                break;
            }
        }
    }
    
    if ($image_url) {
        update_post_meta($post_id, '_thumbnail_ext_url', $image_url);
    }
}
add_action('save_post', 'auto_news_set_featured_image', 100);

// 3. Auto embed videos when URL detected
function auto_news_embed_videos($content) {
    // YouTube
    $content = preg_replace(
        '/\s*https?:\/\/(www\.)?youtube\.com\/watch\?v=([a-zA-Z0-9_-]+)\s*/',
        '<div class="video-container"><iframe width="560" height="315" src="https://www.youtube.com/embed/$2" frameborder="0" allowfullscreen></iframe></div>',
        $content
    );
    
    // Vimeo
    $content = preg_replace(
        '/\s*https?:\/\/vimeo\.com\/([0-9]+)\s*/',
        '<div class="video-container"><iframe src="https://player.vimeo.com/video/$1" width="640" height="360" frameborder="0" allowfullscreen></iframe></div>',
        $content
    );
    
    return $content;
}
add_filter('the_content', 'auto_news_embed_videos');
// server.js
const express = require('express');
const axios = require('axios');
const cheerio = require('cheerio');
const mongoose = require('mongoose');
const app = express();

// Connect to MongoDB
mongoose.connect('mongodb://localhost:27017/autonews', {useNewUrlParser: true});

// News Schema
const newsSchema = new mongoose.Schema({
    title: String,
    content: String,
    source: String,
    imageUrl: String,
    videoUrl: String,
    category: String,
    publishedAt: { type: Date, default: Date.now }
});
const News = mongoose.model('News', newsSchema);

// News API Config
const NEWS_SOURCES = [
    {
        name: 'Reuters',
        url: 'https://www.reuters.com/pf/api/v3/content/fetch/articles-by-section-alias-or-id',
        params: {
            "section_id": "/world",
            "size": 10,
            "website": "reuters"
        }
    },
    // Add more sources
];

// Fetch news function
async function fetchNews() {
    for (const source of NEWS_SOURCES) {
        try {
            const response = await axios.get(source.url, { params: source.params });
            const articles = response.data.result.articles;
            
            for (const article of articles) {
                // Check if news already exists
                const exists = await News.findOne({ title: article.title });
                if (!exists) {
                    // Extract image and video
                    const $ = cheerio.load(article.content);
                    const imageUrl = $('img').first().attr('src') || article.thumbnail;
                    const videoUrl = $('iframe[src*="youtube"]').first().attr('src') || 
                                    $('iframe[src*="vimeo"]').first().attr('src');
                    
                    // Save to DB
                    await News.create({
                        title: article.title,
                        content: article.content,
                        source: source.name,
                        imageUrl,
                        videoUrl,
                        category: article.section || 'General'
                    });
                }
            }
        } catch (error) {
            console.error(`Error fetching from ${source.name}:`, error.message);
        }
    }
}

// Schedule fetching every hour
setInterval(fetchNews, 60 * 60 * 1000);
fetchNews(); // Initial fetch

// API Endpoints
app.get('/api/news', async (req, res) => {
    const { category, limit = 20 } = req.query;
    const query = category ? { category } : {};
    const news = await News.find(query).sort({ publishedAt: -1 }).limit(parseInt(limit));
    res.json(news);
});

app.listen(3000, () => console.log('Server running on port 3000'));
// src/App.js
import React, { useState, useEffect } from 'react';
import axios from 'axios';
import './App.css';

function App() {
    const [news, setNews] = useState([]);
    const [loading, setLoading] = useState(true);
    const [category, setCategory] = useState('all');

    useEffect(() => {
        const fetchNews = async () => {
            try {
                setLoading(true);
                const url = category === 'all' 
                    ? '/api/news' 
                    : `/api/news?category=${category}`;
                const response = await axios.get(url);
                setNews(response.data);
            } catch (error) {
                console.error('Error fetching news:', error);
            } finally {
                setLoading(false);
            }
        };
        
        fetchNews();
    }, [category]);

    return (
        <div className="App">
            <header>
                <h1>Auto News</h1>
                <div className="categories">
                    {['all', 'world', 'technology', 'business', 'sports'].map((cat) => (
                        <button 
                            key={cat}
                            className={category === cat ? 'active' : ''}
                            onClick={() => setCategory(cat)}
                        >
                            {cat.charAt(0).toUpperCase() + cat.slice(1)}
                        </button>
                    ))}
                </div>
            </header>
            
            <main>
                {loading ? (
                    <div className="loading">Loading news...</div>
                ) : (
                    <div className="news-grid">
                        {news.map((item) => (
                            <article key={item._id} className="news-card">
                                {item.imageUrl && (
                                    <img src={item.imageUrl} alt={item.title} />
                                )}
                                <h2>{item.title}</h2>
                                <p>{item.content.substring(0, 150)}...</p>
                                {item.videoUrl && (
                                    <div className="video-container">
                                        <iframe 
                                            src={item.videoUrl} 
                                            title={item.title}
                                            allowFullScreen
                                        />
                                    </div>
                                )}
                                <div className="meta">
                                    <span>{item.source}</span>
                                    <span>{new Date(item.publishedAt).toLocaleDateString()}</span>
                                </div>
                            </article>
                        ))}
                    </div>
                )}
            </main>
        </div>
    );
}

export default App;
/* src/App.css */
.App {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
}

header {
    margin-bottom: 30px;
    text-align: center;
}

.categories {
    display: flex;
    justify-content: center;
    gap: 10px;
    margin-top: 20px;
}

.categories button {
    padding: 8px 16px;
    border: none;
    background: #f0f0f0;
    cursor: pointer;
    border-radius: 4px;
}

.categories button.active {
    background: #007bff;
    color: white;
}

.news-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 25px;
}

.news-card {
    border: 1px solid #ddd;
    border-radius: 8px;
    overflow: hidden;
    transition: transform 0.2s;
}

.news-card:hover {
    transform: translateY(-5px);
    box-shadow: 0 5px 15px rgba(0,0,0,0.1);
}

.news-card img {
    width: 100%;
    height: 180px;
    object-fit: cover;
}

.news-card h2 {
    font-size: 1.2rem;
    margin: 15px;
}

.news-card p {
    margin: 0 15px 15px;
    color: #555;
}

.video-container {
    position: relative;
    padding-bottom: 56.25%;
    height: 0;
    margin: 15px;
}

.video-container iframe {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}

.meta {
    display: flex;
    justify-content: space-between;
    padding: 15px;
    color: #777;
    font-size: 0.9rem;
    border-top: 1px solid #eee;
}

.loading {
    text-align: center;
    padding: 50px;
    font-size: 1.2rem;
    color: #666;
}

@media (max-width: 768px) {
    .news-grid {
        grid-template-columns: 1fr 1fr;
    }
}

@media (max-width: 480px) {
    .news-grid {
        grid-template-columns: 1fr;
    }
    
    .categories {
        flex-wrap: wrap;
    }
}/* src/App.css */
.App {
    max-width: 1200px;
    margin: 0 auto;
    padding: 20px;
}

header {
    margin-bottom: 30px;
    text-align: center;
}

.categories {
    display: flex;
    justify-content: center;
    gap: 10px;
    margin-top: 20px;
}

.categories button {
    padding: 8px 16px;
    border: none;
    background: #f0f0f0;
    cursor: pointer;
    border-radius: 4px;
}

.categories button.active {
    background: #007bff;
    color: white;
}

.news-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
    gap: 25px;
}

.news-card {
    border: 1px solid #ddd;
    border-radius: 8px;
    overflow: hidden;
    transition: transform 0.2s;
}

.news-card:hover {
    transform: translateY(-5px);
    box-shadow: 0 5px 15px rgba(0,0,0,0.1);
}

.news-card img {
    width: 100%;
    height: 180px;
    object-fit: cover;
}

.news-card h2 {
    font-size: 1.2rem;
    margin: 15px;
}

.news-card p {
    margin: 0 15px 15px;
    color: #555;
}

.video-container {
    position: relative;
    padding-bottom: 56.25%;
    height: 0;
    margin: 15px;
}

.video-container iframe {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}

.meta {
    display: flex;
    justify-content: space-between;
    padding: 15px;
    color: #777;
    font-size: 0.9rem;
    border-top: 1px solid #eee;
}

.loading {
    text-align: center;
    padding: 50px;
    font-size: 1.2rem;
    color: #666;
}

@media (max-width: 768px) {
    .news-grid {
        grid-template-columns: 1fr 1fr;
    }
}

@media (max-width: 480px) {
    .news-grid {
        grid-template-columns: 1fr;
    }
    
    .categories {
        flex-wrap: wrap;
    }
}
