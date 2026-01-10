const CACHE_NAME = 'simple-navi-v0.1'; // バージョンを更新
const urlsToCache = [
  './',
  'index.html',
  'manifest.json',
  'sw.js',
  'icon-192.png',
  'https://unpkg.com/leaflet@1.9.4/dist/leaflet.css',
  'https://unpkg.com/leaflet@1.9.4/dist/leaflet.js'
];

// --- 1. インストール時に基本ファイルをキャッシュ ---
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      console.log('Core files caching...');
      return cache.addAll(urlsToCache);
    })
  );
  self.skipWaiting();
});

// --- 2. 更新通知用のメッセージ受け取り ---
self.addEventListener('message', (event) => {
  if (event.data && event.data.type === 'SKIP_WAITING') {
    self.skipWaiting();
  }
});

// --- 3. 古いキャッシュの削除 ---
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames.map((cacheName) => {
          if (cacheName !== CACHE_NAME) {
            console.log('Deleting old cache:', cacheName);
            return caches.delete(cacheName);
          }
        })
      );
    })
  );
  self.clients.claim();
});

// --- 4. ネットワーク優先 + 自動キャッシュ（地図タイル対応） ---
self.addEventListener('fetch', (event) => {
  if (!event.request.url.startsWith('http')) return;

  event.respondWith(
    caches.match(event.request).then((cachedResponse) => {
      // キャッシュがあればそれを即座に返す
      if (cachedResponse) {
        return cachedResponse;
      }

      // キャッシュにない場合、ネットワークから取得
      return fetch(event.request).then((networkResponse) => {
        // 地理院地図のタイル画像(png)などは、取得時に自動でキャッシュに保存する
        if (event.request.url.includes('cyberjapandata.gsi.go.jp')) {
          const responseToCache = networkResponse.clone();
          caches.open(CACHE_NAME).then((cache) => {
            cache.put(event.request, responseToCache);
          });
        }
        return networkResponse;
      }).catch(() => {
        // オフラインかつキャッシュなしの場合
        return new Response('Offline resource not available');
      });
    })
  );
});