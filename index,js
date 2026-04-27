const express = require('express');
const cheerio = require('cheerio');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(express.json());

function buildEbayUrl(keywords, categoryId, maxResults) {
  const base = 'https://www.ebay.com/sch/i.html';
  const params = new URLSearchParams({
    _nkw: keywords,
    LH_Sold: '1',
    LH_Complete: '1',
    _ipg: maxResults || '60',
  });
  if (categoryId && categoryId !== '0') {
    params.append('_sacat', categoryId);
  }
  return `${base}?${params.toString()}`;
}

function parseListings(html, excludedKeywords) {
  const $ = cheerio.load(html);
  const listings = [];
  const excluded = excludedKeywords
    ? excludedKeywords.toLowerCase().split(' ')
    : [];

  $('.s-item').each((i, el) => {
    const title = $(el).find('.s-item__title span').first().text().trim();
    const priceText = $(el).find('.s-item__price').first().text().trim();
    const date = $(el).find('.s-item__ended-date').first().text().trim();
    const link = $(el).find('.s-item__link').first().attr('href') || '';

    if (!title || title === 'Shop on eBay') return;
    if (excluded.some(kw => kw && title.toLowerCase().includes(kw))) return;

    const priceMatch = priceText.match(/\$([0-9,]+\.?\d*)/);
    if (!priceMatch) return;

    const price = parseFloat(priceMatch[1].replace(',', ''));
    if (!price || price <= 0) return;

    listings.push({ title, price, date, link });
  });

  return listings;
}

function removeOutliers(listings) {
  if (listings.length < 4) return listings;
  const prices = listings.map(l => l.price).sort((a, b) => a - b);
  const q1 = prices[Math.floor(prices.length * 0.25)];
  const q3 = prices[Math.floor(prices.length * 0.75)];
  const iqr = q3 - q1;
  const lower = q1 - 1.5 * iqr;
  const upper = q3 + 1.5 * iqr;
  return listings.filter(l => l.price >= lower && l.price <= upper);
}

function calculateStats(listings) {
  const prices = listings.map(l => l.price).sort((a, b) => a - b);
  const avg = prices.reduce((a, b) => a + b, 0) / prices.length;
  const median = prices[Math.floor(prices.length / 2)];
  return {
    average_price: Math.round(avg * 100) / 100,
    median_price: Math.round(median * 100) / 100,
    min_price: prices[0],
    max_price: prices[prices.length - 1],
  };
}

async function launchBrowser() {
  if (process.env.RAILWAY_ENVIRONMENT) {
    console.log('Launching stealth Puppeteer with @sparticuz/chromium (Railway)');
    const puppeteerExtra = require('puppeteer-extra');
    const StealthPlugin = require('puppeteer-extra-plugin-stealth');
    const chromium = require('@sparticuz/chromium');
    puppeteerExtra.use(StealthPlugin());
    return await puppeteerExtra.launch({
      args: chromium.args,
      defaultViewport: chromium.defaultViewport,
      executablePath: await chromium.executablePath(),
      headless: true,
    });
  } else {
    console.log('Launching stealth Puppeteer with local Chromium (dev)');
    const puppeteerExtra = require('puppeteer-extra');
    const StealthPlugin = require('puppeteer-extra-plugin-stealth');
    puppeteerExtra.use(StealthPlugin());
    return await puppeteerExtra.launch({
      headless: true,
      args: [
        '--no-sandbox',
        '--disable-setuid-sandbox',
        '--disable-blink-features=AutomationControlled',
      ],
    });
  }
}

async function fetchEbayHtml(url) {
  console.log('Launching stealth browser...');
  const browser = await launchBrowser();

  try {
    const page = await browser.newPage();

    // Set a realistic viewport
    await page.setViewport({ width: 1366, height: 768 });

    // Set realistic user agent
    await page.setUserAgent(
      'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36'
    );

    // Set extra headers to look like a real browser
    await page.setExtraHTTPHeaders({
      'Accept-Language': 'en-US,en;q=0.9',
      'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8',
      'Accept-Encoding': 'gzip, deflate, br',
      'Connection': 'keep-alive',
      'Upgrade-Insecure-Requests': '1',
    });

    console.log('Navigating to:', url);
    await page.goto(url, {
      waitUntil: 'networkidle2',
      timeout: 30000,
    });

    // Wait for listings to appear
    await page.waitForSelector('.s-item', { timeout: 10000 })
      .catch(() => console.log('Warning: .s-item not found — may be blocked or no results'));

    const html = await page.content();
    console.log('HTML length:', html.length);

    // Check if we got a real page or a block page
    if (html.length < 5000) {
      console.log('HTML preview:', html.substring(0, 500));
      throw new Error('Got a very short response — likely blocked');
    }

    return html;
  } finally {
    await browser.close();
  }
}

app.post('/findCompletedItems', async (req, res) => {
  const {
    keywords,
    excluded_keywords,
    category_id,
    remove_outliers,
    max_search_results,
  } = req.body;

  if (!keywords) {
    return res.status(400).json({ error: 'keywords is required' });
  }

  try {
    const url = buildEbayUrl(keywords, category_id, max_search_results);
    console.log('\n--- New Request ---');
    console.log('Keywords:', keywords);
    console.log('URL:', url);

    const html = await fetchEbayHtml(url);
    let listings = parseListings(html, excluded_keywords);
    console.log('Raw listings found:', listings.length);

    if (listings.length === 0) {
      return res.json({
        success: true,
        average_price: null,
        median_price: null,
        min_price: null,
        max_price: null,
        results: 0,
        products: [],
        message: 'No sold listings found',
        query_url: url,
      });
    }

    if (remove_outliers) {
      listings = removeOutliers(listings);
      console.log('Listings after outlier removal:', listings.length);
    }

    const stats = calculateStats(listings);
    console.log('Average price:', stats.average_price);

    return res.json({
      success: true,
      ...stats,
      results: listings.length,
      products: listings.slice(0, 10),
      query_url: url,
    });

  } catch (err) {
    console.error('Error:', err.message);
    return res.status(500).json({
      success: false,
      error: err.message,
    });
  }
});

app.get('/', (req, res) => {
  res.json({
    status: 'Vaultd Pricer is running',
    environment: process.env.RAILWAY_ENVIRONMENT ? 'railway' : 'local',
  });
});

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
  console.log(`Vaultd Pricer running on port ${PORT}`);
});