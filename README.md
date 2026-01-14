import React, { useState, useEffect, useRef, useCallback } from 'react';
import {
  Search,
  RefreshCw,
  Newspaper,
  TrendingUp,
  Globe,
  Zap,
  ThumbsUp,
  AlertCircle,
  ThumbsDown,
  Youtube,
  Settings,
  CheckCircle,
  XCircle
} from 'lucide-react';

/*
  Notes on changes:
  - Added AbortController support to cancel in-flight fetches.
  - Debounced search input to reduce requests.
  - Pass category consistently into YouTube fetch (bugfix).
  - Use Promise.allSettled / graceful handling so one failing source doesn't fail everything.
  - Deduplicate articles by URL/title, sort by date.
  - Use stable keys for list items (avoid index).
  - Lazy-load images and handle errors by hiding image or using a placeholder.
  - Persist API keys to localStorage (client-side storage is still insecure for private keys —
    see recommendation to move keys to backend).
  - Minor accessibility improvements (aria-labels).
  - Use onKeyDown rather than deprecated onKeyPress.
*/

export default function NewsAggregator() {
  const [articles, setArticles] = useState([]);
  const [loading, setLoading] = useState(false);
  const [searchQuery, setSearchQuery] = useState('');
  const [category, setCategory] = useState('general');
  const [lastUpdate, setLastUpdate] = useState(new Date());
  const [showSettings, setShowSettings] = useState(false);
  const [apiKeys, setApiKeys] = useState({
    newsapi: '',
    youtube: ''
  });
  const [apiStatus, setApiStatus] = useState({
    newsapi: false,
    youtube: false,
    reddit: false,
    rss: true
  });

  const categories = ['general', 'technology', 'business', 'science', 'health', 'sports', 'entertainment'];

  const debounceRef = useRef(null);
  const abortControllerRef = useRef(null);

  // Load API keys from localStorage (if any) on mount
  useEffect(() => {
    try {
      const saved = localStorage.getItem('newsflow_api_keys');
      if (saved) setApiKeys(JSON.parse(saved));
    } catch (e) {
      // ignore
    }
    loadDemoData();
    // cleanup on unmount
    return () => {
      if (abortControllerRef.current) abortControllerRef.current.abort();
    };
  }, []);

  // Basic sentiment analyzer (keeps your original approach but can be improved/replaced by a proper NLP library)
  const analyzeSentiment = (title, description) => {
    const text = `${title || ''} ${description || ''}`.toLowerCase();

    const positiveWords = ['breakthrough', 'success', 'growth', 'win', 'achievement', 'advance', 'improve', 'gain', 'rise', 'surge', 'record', 'milestone', 'innovation', 'agreement', 'victory', 'boost', 'profit', 'recover', 'expand', 'celebrates'];
    const negativeWords = ['crisis', 'crash', 'fail', 'loss', 'death', 'disaster', 'decline', 'fall', 'collapse', 'threat', 'danger', 'violence', 'attack', 'war', 'conflict', 'risk', 'concern', 'scandal', 'lawsuit', 'threatens'];

    let positiveCount = 0;
    let negativeCount = 0;

    positiveWords.forEach(word => { if (text.includes(word)) positiveCount++; });
    negativeWords.forEach(word => { if (text.includes(word)) negativeCount++; });

    if (positiveCount > negativeCount) return 'positive';
    if (negativeCount > positiveCount) return 'negative';
    return 'neutral';
  };

  const getSentimentStyle = (sentiment) => {
    switch (sentiment) {
      case 'positive':
        return { textColor: 'text-blue-400', icon: ThumbsUp, label: 'Positive' };
      case 'negative':
        return { textColor: 'text-red-400', icon: ThumbsDown, label: 'Negative' };
      default:
        return { textColor: 'text-yellow-500', icon: AlertCircle, label: 'Unclear' };
    }
  };

  // Utility to dedupe articles by url or title+publishedAt
  const dedupeArticles = (arr) => {
    const map = new Map();
    for (const a of arr) {
      const key = (a.url && a.url !== '#') ? a.url : `${a.title}::${a.publishedAt || ''}`;
      if (!map.has(key)) map.set(key, a);
    }
    return Array.from(map.values());
  };

  // Fetch helpers accept a signal for cancellation
  const fetchNewsAPI = async (query = '', cat = category, signal) => {
    if (!apiKeys.newsapi) return [];
    try {
      const endpoint = query
        ? `https://newsapi.org/v2/everything?q=${encodeURIComponent(query)}&sortBy=publishedAt&language=en&pageSize=20`
        : `https://newsapi.org/v2/top-headlines?category=${encodeURIComponent(cat)}&country=us&language=en&pageSize=20`;

      const response = await fetch(endpoint, {
        headers: { 'X-Api-Key': apiKeys.newsapi },
        signal
      });

      if (response.ok) {
        const data = await response.json();
        setApiStatus(prev => ({ ...prev, newsapi: true }));
        return (data.articles || []).map(article => ({ ...article, source: { name: article.source.name, platform: 'NewsAPI' } }));
      } else {
        setApiStatus(prev => ({ ...prev, newsapi: false }));
      }
    } catch (error) {
      if (error.name !== 'AbortError') console.error('NewsAPI error:', error);
      setApiStatus(prev => ({ ...prev, newsapi: false }));
    }
    return [];
  };

  const fetchYouTube = async (query = '', cat = category, signal) => {
    if (!apiKeys.youtube) return [];
    try {
      const searchTerm = query || `${cat} news` || 'news';
      const response = await fetch(
        `https://www.googleapis.com/youtube/v3/search?part=snippet&q=${encodeURIComponent(searchTerm)}&type=video&maxResults=10&order=date&relevanceLanguage=en&key=${apiKeys.youtube}`,
        { signal }
      );

      if (response.ok) {
        const data = await response.json();
        setApiStatus(prev => ({ ...prev, youtube: true }));
        return (data.items || []).map(item => ({
          title: item.snippet.title,
          description: item.snippet.description,
          source: { name: item.snippet.channelTitle, platform: 'YouTube' },
          publishedAt: item.snippet.publishedAt,
          url: `https://www.youtube.com/watch?v=${item.id.videoId}`,
          urlToImage: item.snippet.thumbnails?.medium?.url
        }));
      } else {
        setApiStatus(prev => ({ ...prev, youtube: false }));
      }
    } catch (error) {
      if (error.name !== 'AbortError') console.error('YouTube error:', error);
      setApiStatus(prev => ({ ...prev, youtube: false }));
    }
    return [];
  };

  const fetchReddit = async (query = '', signal) => {
    try {
      const subredditPath = query ? `search.json?q=${encodeURIComponent(query)}&sort=new&limit=10` : 'r/worldnews/hot.json?limit=10';
      const url = query ? `https://www.reddit.com/${subredditPath}` : `https://www.reddit.com/${subredditPath}`;
      const response = await fetch(url, { signal });

      if (response.ok) {
        const data = await response.json();
        setApiStatus(prev => ({ ...prev, reddit: true }));
        return (data.data?.children || []).map(item => ({
          title: item.data.title,
          description: item.data.selftext || item.data.url,
          source: { name: `r/${item.data.subreddit}`, platform: 'Reddit' },
          publishedAt: new Date(item.data.created_utc * 1000).toISOString(),
          url: `https://reddit.com${item.data.permalink}`,
          urlToImage: item.data.thumbnail && item.data.thumbnail.startsWith('http') ? item.data.thumbnail : null
        }));
      } else {
        setApiStatus(prev => ({ ...prev, reddit: false }));
      }
    } catch (error) {
      if (error.name !== 'AbortError') console.error('Reddit error:', error);
      setApiStatus(prev => ({ ...prev, reddit: false }));
    }
    return [];
  };

  const fetchRSSFeeds = async (query = '', signal) => {
    try {
      const feeds = [
        'http://feeds.bbci.co.uk/news/rss.xml',
        'http://rss.cnn.com/rss/cnn_topstories.rss',
        'https://feeds.npr.org/1001/rss.xml'
      ];

      const allArticles = [];
      for (const feedUrl of feeds) {
        try {
          const response = await fetch(`https://api.rss2json.com/v1/api.json?rss_url=${encodeURIComponent(feedUrl)}`, { signal });
          if (response.ok) {
            const data = await response.json();
            if (data.items) {
              const feedArticles = data.items.map(item => ({
                title: item.title,
                description: item.description?.replace(/<[^>]*>/g, '').substring(0, 200),
                source: { name: data.feed?.title || 'RSS', platform: 'RSS' },
                publishedAt: item.pubDate,
                url: item.link,
                urlToImage: item.enclosure?.link || item.thumbnail
              }));
              allArticles.push(...feedArticles);
            }
          }
        } catch (err) {
          if (err.name !== 'AbortError') console.error('RSS feed error:', err);
        }
      }

      setApiStatus(prev => ({ ...prev, rss: allArticles.length > 0 }));
      return allArticles;
    } catch (error) {
      if (error.name !== 'AbortError') console.error('RSS error:', error);
      return [];
    }
  };

  // Main fetch orchestration
  const fetchAllNews = useCallback(async (query = '', cat = category) => {
    // Cancel previous fetches
    if (abortControllerRef.current) abortControllerRef.current.abort();
    const controller = new AbortController();
    abortControllerRef.current = controller;
    const signal = controller.signal;

    setLoading(true);
    try {
      // Use Promise.allSettled so a single source failure doesn't break everything
      const results = await Promise.allSettled([
        fetchNewsAPI(query, cat, signal),
        fetchYouTube(query, cat, signal),
        fetchReddit(query, signal),
        fetchRSSFeeds(query, signal)
      ]);

      const resolvedArticles = results.flatMap(r => (r.status === 'fulfilled' ? r.value : []));

      const deduped = dedupeArticles(resolvedArticles);

      // Sort by date (fallback: keep original order for items without dates)
      deduped.sort((a, b) => {
        const da = new Date(a.publishedAt || 0).getTime();
        const db = new Date(b.publishedAt || 0).getTime();
        return db - da;
      });

      if (deduped.length === 0) {
        loadDemoData();
      } else {
        setArticles(deduped);
        setLastUpdate(new Date());
      }
    } catch (error) {
      if (error.name !== 'AbortError') {
        console.error('Error fetching news:', error);
        loadDemoData();
      }
    } finally {
      setLoading(false);
    }
  }, [apiKeys, category]);

  const loadDemoData = () => {
    const demoArticles = [
      {
        title: "AI Breakthrough: New Model Achieves Human-Level Performance",
        description: "Researchers announce significant advancement in artificial intelligence capabilities, marking a milestone in machine learning development.",
        source: { name: "Tech News Daily", platform: "Demo" },
        publishedAt: new Date().toISOString(),
        url: "#",
        urlToImage: "https://images.unsplash.com/photo-1677442136019-21780ecad995?w=800&q=80"
      },
      {
        title: "Global Markets Crash Amid Economic Crisis Fears",
        description: "Stock markets worldwide show significant decline as major economies face potential recession and financial instability.",
        source: { name: "Financial Times", platform: "Demo" },
        publishedAt: new Date(Date.now() - 3600000).toISOString(),
        url: "#",
        urlToImage: "https://images.unsplash.com/photo-1611974789855-9c2a0a7236a3?w=800&q=80"
      },
      {
        title: "Climate Summit Reaches Historic Agreement",
        description: "World leaders commit to ambitious carbon reduction targets in landmark environmental accord signed at international conference.",
        source: { name: "Global News Network", platform: "Demo" },
        publishedAt: new Date(Date.now() - 7200000).toISOString(),
        url: "#",
        urlToImage: "https://images.unsplash.com/photo-1569163139394-de4798aa62b6?w=800&q=80"
      }
    ];
    setArticles(demoArticles);
    setLastUpdate(new Date());
  };

  // Debounced search
  const handleSearch = (q = searchQuery) => {
    if (debounceRef.current) clearTimeout(debounceRef.current);
    debounceRef.current = setTimeout(() => {
      fetchAllNews(q, category);
    }, 450);
  };

  // Category change should immediately fetch
  const handleCategoryChange = (cat) => {
    setCategory(cat);
    fetchAllNews('', cat);
  };

  // Save API keys to state and localStorage, then fetch
  const saveApiKeys = () => {
    try {
      localStorage.setItem('newsflow_api_keys', JSON.stringify(apiKeys));
    } catch (e) {
      // ignore storage errors
    }
    fetchAllNews(searchQuery, category);
    setShowSettings(false);
  };

  const formatTime = (dateString) => {
    const date = new Date(dateString);
    const now = new Date();
    const diff = Math.floor((now - date) / (1000 * 60)); // minutes
    if (diff < 1) return 'just now';
    if (diff < 60) return `${diff}m ago`;
    if (diff < 1440) return `${Math.floor(diff / 60)}h ago`;
    return `${Math.floor(diff / 1440)}d ago`;
  };

  // Image error handler: hide the image element or set placeholder
  const onImageError = (e) => {
    try {
      e.currentTarget.style.display = 'none';
    } catch (err) {
      // ignore
    }
  };

  return (
    <div className="min-h-screen bg-white">
      {/* Settings Modal */}
      {showSettings && (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50 p-4" role="dialog" aria-modal="true" aria-label="API configuration dialog">
          <div className="bg-white rounded-lg max-w-2xl w-full max-h-[90vh] overflow-y-auto p-6">
            <div className="flex items-center justify-between mb-6">
              <h2 className="text-2xl font-bold text-gray-800">API Configuration</h2>
              <button onClick={() => setShowSettings(false)} className="text-gray-500 hover:text-gray-700" aria-label="Close settings">
                ✕
              </button>
            </div>

            <div className="space-y-6">
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-2">NewsAPI Key (80,000+ news sources)</label>
                <input
                  type="text"
                  value={apiKeys.newsapi}
                  onChange={(e) => setApiKeys({ ...apiKeys, newsapi: e.target.value })}
                  placeholder="Enter your NewsAPI key"
                  className="w-full px-4 py-2 border-2 border-gray-200 rounded-lg focus:outline-none focus:border-blue-400"
                  aria-label="NewsAPI key"
                />
                <p className="text-sm text-gray-500 mt-1">
                  Get free at <a href="https://newsapi.org" target="_blank" rel="noreferrer" className="text-blue-500 hover:underline">newsapi.org</a>
                </p>
              </div>

              <div>
                <label className="block text-sm font-medium text-gray-700 mb-2">YouTube API Key (Video news)</label>
                <input
                  type="text"
                  value={apiKeys.youtube}
                  onChange={(e) => setApiKeys({ ...apiKeys, youtube: e.target.value })}
                  placeholder="Enter your YouTube API key"
                  className="w-full px-4 py-2 border-2 border-gray-200 rounded-lg focus:outline-none focus:border-blue-400"
                  aria-label="YouTube API key"
                />
                <p className="text-sm text-gray-500 mt-1">
                  Get at <a href="https://console.cloud.google.com" target="_blank" rel="noreferrer" className="text-blue-500 hover:underline">Google Cloud Console</a>
                </p>
              </div>

              <div className="bg-blue-50 p-4 rounded-lg">
                <h3 className="font-medium text-gray-800 mb-2">Reddit (No API key needed!)</h3>
                <p className="text-sm text-gray-600">Reddit news is automatically fetched from r/worldnews and r/news without authentication.</p>
              </div>

              <div className="bg-green-50 p-4 rounded-lg">
                <h3 className="font-medium text-gray-800 mb-2">RSS Feeds (Always Active!)</h3>
                <p className="text-sm text-gray-600">BBC, CNN, and NPR feeds are automatically included without any API keys.</p>
              </div>

              <button onClick={saveApiKeys} className="w-full bg-blue-500 hover:bg-blue-600 text-white font-medium py-3 rounded-lg transition-colors">
                Save & Fetch News
              </button>
            </div>
          </div>
        </div>
      )}

      {/* Header */}
      <header className="bg-white sticky top-0 z-40 pb-6 pt-4 border-b border-gray-100">
        <div className="max-w-7xl mx-auto px-4">
          <div className="flex items-center justify-between mb-6">
            <div className="flex items-center gap-3">
              <Zap className="w-10 h-10 text-blue-500" />
              <h1 className="text-3xl font-bold text-gray-800">NewsFlow AI</h1>
            </div>
            <div className="flex items-center gap-4">
              <button onClick={() => setShowSettings(true)} className="flex items-center gap-2 px-4 py-2 text-gray-600 hover:text-gray-800 transition-colors" aria-label="Open API keys">
                <Settings className="w-5 h-5" />
                API Keys
              </button>
              <button onClick={() => fetchAllNews(searchQuery, category)} className="flex items-center gap-2 px-5 py-2 text-blue-500 hover:text-blue-600 transition-colors font-medium" aria-label="Refresh news">
                <RefreshCw className={`w-5 h-5 ${loading ? 'animate-spin' : ''}`} />
                Refresh
              </button>
              <span className="text-sm text-gray-500">{formatTime(lastUpdate.toISOString())}</span>
            </div>
          </div>

          {/* API Status */}
          <div className="flex gap-4 mb-4 text-xs">
            <div className="flex items-center gap-1">
              {apiStatus.newsapi ? <CheckCircle className="w-4 h-4 text-green-500" /> : <XCircle className="w-4 h-4 text-gray-300" />}
              <span className={apiStatus.newsapi ? 'text-green-600' : 'text-gray-400'}>NewsAPI</span>
            </div>
            <div className="flex items-center gap-1">
              {apiStatus.youtube ? <CheckCircle className="w-4 h-4 text-green-500" /> : <XCircle className="w-4 h-4 text-gray-300" />}
              <span className={apiStatus.youtube ? 'text-green-600' : 'text-gray-400'}>YouTube</span>
            </div>
            <div className="flex items-center gap-1">
              {apiStatus.reddit ? <CheckCircle className="w-4 h-4 text-green-500" /> : <XCircle className="w-4 h-4 text-gray-300" />}
              <span className={apiStatus.reddit ? 'text-green-600' : 'text-gray-400'}>Reddit</span>
            </div>
            <div className="flex items-center gap-1">
              {apiStatus.rss ? <CheckCircle className="w-4 h-4 text-green-500" /> : <XCircle className="w-4 h-4 text-gray-300" />}
              <span className={apiStatus.rss ? 'text-green-600' : 'text-gray-400'}>RSS Feeds</span>
            </div>
          </div>

          {/* Search Bar */}
          <div className="mb-6">
            <div className="relative">
              <Search className="absolute left-4 top-1/2 -translate-y-1/2 w-5 h-5 text-gray-400" />
              <input
                type="text"
                value={searchQuery}
                onChange={(e) => { setSearchQuery(e.target.value); }}
                onKeyDown={(e) => { if (e.key === 'Enter') { handleSearch(searchQuery); } }}
                onBlur={() => handleSearch(searchQuery)}
                placeholder="Search across all platforms..."
                className="w-full pl-12 pr-4 py-3 bg-white text-gray-800 placeholder-gray-400 focus:outline-none border-b-2 border-gray-200 focus:border-blue-400 transition-colors"
                aria-label="Search news"
              />
            </div>
          </div>

          {/* Categories */}
          <div className="flex gap-6 overflow-x-auto pb-2">
            {categories.map((cat) => (
              <button
                key={cat}
                onClick={() => handleCategoryChange(cat)}
                className={`text-sm font-medium transition-colors whitespace-nowrap pb-2 ${category === cat ? 'text-blue-500 border-b-2 border-blue-500' : 'text-gray-600 hover:text-gray-800'}`}
                aria-pressed={category === cat}
              >
                {cat.charAt(0).toUpperCase() + cat.slice(1)}
              </button>
            ))}
          </div>
        </div>
      </header>

      {/* Main Content */}
      <main className="max-w-7xl mx-auto px-4 py-8">
        {loading ? (
          <div className="flex items-center justify-center py-20">
            <RefreshCw className="w-12 h-12 text-blue-500 animate-spin" />
          </div>
        ) : (
          <div className="space-y-8">
            {articles.map((article) => {
              const sentiment = analyzeSentiment(article.title, article.description || '');
              const style = getSentimentStyle(sentiment);
              const SentimentIcon = style.icon;
              const key = article.url && article.url !== '#' ? article.url : `${article.title}::${article.publishedAt}`;

              return (
                <article key={key} className="group cursor-pointer py-6 border-b border-gray-100 hover:bg-gray-50 transition-colors px-4 -mx-4">
                  <div className="flex gap-6">
                    {article.urlToImage && (
                      <div className="flex-shrink-0 w-48 h-32 overflow-hidden">
                        <img
                          src={article.urlToImage}
                          alt={article.title || 'article image'}
                          className="w-full h-full object-cover group-hover:scale-105 transition-transform duration-300"
                          onError={onImageError}
                          loading="lazy"
                        />
                      </div>
                    )}
                    <div className="flex-1">
                      <div className="flex items-center gap-3 mb-2">
                        <span className="text-sm font-medium text-gray-700">{article.source?.name}</span>
                        <span className="text-xs text-gray-400 bg-gray-100 px-2 py-1 rounded">{article.source?.platform}</span>
                        <span className="text-gray-300">•</span>
                        <span className="text-sm text-gray-400">{formatTime(article.publishedAt)}</span>
                        <span className="text-gray-300">•</span>
                        <div className="flex items-center gap-1">
                          <SentimentIcon className={`w-4 h-4 ${style.textColor}`} />
                          <span className={`text-xs font-medium ${style.textColor}`}>{style.label}</span>
                        </div>
                      </div>
                      <h2 className={`text-2xl font-bold ${style.textColor} mb-2 group-hover:opacity-80 transition-opacity`}>{article.title}</h2>
                      <p className="text-gray-600 text-base mb-3 line-clamp-2">{article.description}</p>
                      <a href={article.url} target="_blank" rel="noopener noreferrer" className={`inline-flex items-center gap-2 ${style.textColor} hover:opacity-70 text-sm font-medium`}>
                        Read Full Story
                        <TrendingUp className="w-4 h-4" />
                      </a>
                    </div>
                  </div>
                </article>
              );
            })}
          </div>
        )}

        {/* Quick Setup Guide */}
        <div className="mt-12 pt-8 border-t-2 border-gray-100">
          <h3 className="text-xl font-bold text-gray-800 mb-4">🚀 Quick Setup Guide</h3>
          <div className="space-y-4 text-gray-600">
            <div className="flex items-start gap-3">
              <span className="font-bold text-blue-500">1.</span>
              <div>
                <p className="font-medium">Click "API Keys" button above</p>
                <p className="text-sm">Configure your API keys for NewsAPI and YouTube</p>
              </div>
            </div>
            <div className="flex items-start gap-3">
              <span className="font-bold text-green-500">2.</span>
              <div>
                <p className="font-medium">Reddit & RSS work automatically!</p>
                <p className="text-sm">No setup needed - starts fetching immediately</p>
              </div>
            </div>
            <div className="flex items-start gap-3">
              <span className="font-bold text-purple-500">3.</span>
              <div>
                <p className="font-medium">Deploy to your server</p>
                <p className="text-sm">Upload this React app to Vercel, Netlify, or any hosting service</p>
              </div>
            </div>
          </div>
        </div>
      </main>
    </div>
  );
}
