<!DOCTYPE html>
<html lang="en"><head>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/html-to-image/1.11.13/html-to-image.min.js" integrity="sha512-iZ2ORl595Wx6miw+GuadDet4WQbdSWS3JLMoNfY8cRGoEFy6oT3G9IbcrBeL6AfkgpA51ETt/faX6yLV+/gFJg==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>
    <script>
      (function() {
        const originalConsole = window.console;
        window.console = {
          log: (...args) => {
            originalConsole.log(...args);
            window.parent.postMessage({ type: 'console', message: args.join(' ') }, '*');
          },
          error: (...args) => {
            originalConsole.error(...args);
            window.parent.postMessage({ type: 'console', message: 'Error: ' + args.join(' ') }, '*');
          },
          warn: (...args) => {
            originalConsole.warn(...args);
            window.parent.postMessage({ type: 'console', message: 'Warning: ' + args.join(' ') }, '*');
          }
        };

        let requestId = 0;
        let callbacksMap = new Map();
        let streamControllers = new Map();
        
        window.claude = {
          complete: (prompt) => {
            return new Promise((resolve, reject) => {
              const id = requestId++;
              callbacksMap.set(id, { resolve, reject });
              window.parent.postMessage({ type: 'claudeComplete', id, prompt }, '*');
            });
          }
        };

        window.storage = {
          get: (key, shared = false) => {
            return new Promise((resolve, reject) => {
              const id = requestId++;
              callbacksMap.set(id, { resolve, reject });
              window.parent.postMessage({ type: 'storageGet', id, key, shared }, '*');
            });
          },
          set: (key, value, shared = false) => {
            return new Promise((resolve, reject) => {
              const id = requestId++;
              callbacksMap.set(id, { resolve, reject });
              window.parent.postMessage({ type: 'storageSet', id, key, value, shared }, '*');
            });
          },
          delete: (key, shared = false) => {
            return new Promise((resolve, reject) => {
              const id = requestId++;
              callbacksMap.set(id, { resolve, reject });
              window.parent.postMessage({ type: 'storageDelete', id, key, shared }, '*');
            });
          },
          list: (prefix, shared = false) => {
            return new Promise((resolve, reject) => {
              const id = requestId++;
              callbacksMap.set(id, { resolve, reject });
              window.parent.postMessage({ type: 'storageList', id, prefix, shared }, '*');
            });
          }
        };

        let pendingBlobs = new Map();
        URL.createObjectURL = (blob) => {
          // Store the blob and create an ID and URL for it
          const blobId = `blob-${Date.now()}-${Math.random()}`;
          pendingBlobs.set(blobId, blob);
          return `blob-request://${blobId}`;
        };

        URL.revokeObjectURL = (url) => {
          // Remove the blob from our store
          const blobId = url.replace("blob-request://", "");
          pendingBlobs.delete(blobId);
        };

        const getBlobFromURL = (url) => {
          const blobId = url.replace("blob-request://", "");
          return pendingBlobs.get(blobId);
        };

        // Override global fetch with streaming support
        window.fetch = (url, init = {}) => {
          return new Promise((resolve, reject) => {
            const id = requestId++;
            const channelId = `fetch-${id}-${Date.now()}`;
            
            callbacksMap.set(id, { 
              resolve: (response) => {
                // Create a ReadableStream for the response body
                const stream = new ReadableStream({
                  start(controller) {
                    streamControllers.set(channelId, controller);
                  },
                  cancel() {
                    streamControllers.delete(channelId);
                  }
                });
                
                // Create and return the Response with the stream
                resolve(new Response(stream, {
                  status: response.status,
                  statusText: response.statusText,
                  headers: response.headers
                }));
              },
              reject,
              channelId
            });
            
            window.parent.postMessage({
              type: 'proxyFetch',
              id,
              url,
              init,
              channelId
            }, '*');
          });
        };

        window.addEventListener('message', async (event) => {
          if (event.data.type === 'takeScreenshot') {
            const rootElement = document.getElementById('artifacts-component-root-html');
            if (!rootElement) {
              window.parent.postMessage({
                type: 'screenshotError',
                error: new Error('Root element not found'),
              }, '*');
            }
            const screenshot = await htmlToImage.toPng(rootElement, {
              imagePlaceholder:
                "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAAAXNSR0IArs4c6QAAAA1JREFUGFdjePDgwX8ACOQDoNsk0PMAAAAASUVORK5CYII=",
            });
            window.parent.postMessage({
              type: 'screenshotData',
              data: screenshot,
            }, '*');
          } else if (event.data.type === 'claudeComplete') {
            const callback = callbacksMap.get(event.data.id);
            if (event.data.error) {
              callback.reject(new Error(event.data.error));
            } else {
              callback.resolve(event.data.completion);
            }
            callbacksMap.delete(event.data.id);
          } else if (event.data.type === 'proxyFetchResponse') {
            const callback = callbacksMap.get(event.data.id);
            if (event.data.error) {
              callback.reject(new Error(event.data.error));
              callbacksMap.delete(event.data.id);
            } else {
              // Initial response with headers, status, etc.
              callback.resolve({
                status: event.data.status,
                statusText: event.data.statusText,
                headers: event.data.headers
              });
              // Don't delete the callback yet if streaming
              if (!event.data.body) {
                callbacksMap.delete(event.data.id);
              }
            }
          } else if (event.data.type === 'proxyFetchStream') {
            // Handle streaming data chunks
            const controller = streamControllers.get(event.data.channelId);
            if (controller) {
              if (event.data.error) {
                controller.error(new Error(event.data.error));
                streamControllers.delete(event.data.channelId);
              } else if (event.data.done) {
                controller.close();
                streamControllers.delete(event.data.channelId);
                // Clean up the callback
                const callback = Array.from(callbacksMap.entries()).find(
                  ([_, value]) => value.channelId === event.data.channelId
                );
                if (callback) {
                  callbacksMap.delete(callback[0]);
                }
              } else if (event.data.chunk) {
                controller.enqueue(new Uint8Array(event.data.chunk));
              }
            }
          } else if (event.data.type === 'storageGet') {
            const callback = callbacksMap.get(event.data.id);
            if (event.data.error) {
              callback.reject(new Error(event.data.error));
            } else {
              callback.resolve(event.data.result);
            }
            callbacksMap.delete(event.data.id);
          } else if (event.data.type === 'storageSet') {
            const callback = callbacksMap.get(event.data.id);
            if (event.data.error) {
              callback.reject(new Error(event.data.error));
            } else {
              callback.resolve(event.data.result);
            }
            callbacksMap.delete(event.data.id);
          } else if (event.data.type === 'storageDelete') {
            const callback = callbacksMap.get(event.data.id);
            if (event.data.error) {
              callback.reject(new Error(event.data.error));
            } else {
              callback.resolve(event.data.result);
            }
            callbacksMap.delete(event.data.id);
          } else if (event.data.type === 'storageList') {
            const callback = callbacksMap.get(event.data.id);
            if (event.data.error) {
              callback.reject(new Error(event.data.error));
            } else {
              callback.resolve(event.data.result);
            }
            callbacksMap.delete(event.data.id);
          }
        });

        window.addEventListener('click', (event) => {
          const isEl = event.target instanceof HTMLElement;
          if (!isEl) return;
    
          // find ancestor links
          const linkEl = event.target.closest("a");
          if (!linkEl || !linkEl.href) return;
    
          event.preventDefault();
          event.stopImmediatePropagation();
    
          if (linkEl.href.startsWith("blob-request:")) {
            const blob = getBlobFromURL(linkEl.href);
            if (!blob) return;
            void blob.arrayBuffer().then((data) => {
              window.parent.postMessage({
                type: "downloadFile",
                filename: linkEl.download,
                data,
                mimeType: blob.type || "application/octet-stream",
              });
            });
          } else if (linkEl.href.startsWith("data:")) {
            const [header, base64Data] = linkEl.href.split(",");
            const mimeMatch = header.match(/data:([^;]+)/);
            const mimeType = mimeMatch ? mimeMatch[1] : "application/octet-stream";
            const binaryString = atob(base64Data);
            const data = Uint8Array.from(binaryString, (c) =>
              c.charCodeAt(0),
            ).buffer;
            window.parent.postMessage({
              type: "downloadFile",
              filename: linkEl.download,
              data,
              mimeType,
            });
          } else {
            let linkUrl;
            try {
              linkUrl = new URL(linkEl.href);
            } catch (error) {
              return;
            }
    
            if (linkUrl.hostname === window.location.hostname) return;
      
            window.parent.postMessage({
              type: 'openExternal',
              href: linkEl.href,
            }, '*');
          }
      });

        const originalOpen = window.open;
        window.open = function (url) {
          window.parent.postMessage({
            type: "openExternal",
            href: url,
          }, "*");
        };

        window.addEventListener('error', (event) => {
          window.parent.postMessage({ type: 'console', message: 'Uncaught Error: ' + event.message }, '*');
        });
      })();
    </script>
  
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1.0,maximum-scale=1.0,user-scalable=no,viewport-fit=cover">
<meta name="theme-color" content="#0a0a1a">
<title>ELJ2030 Quiz</title>
<script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
<style>
@import url('https://fonts.googleapis.com/css2?family=Noto+Sans+JP:wght@400;700;900&family=Inter:wght@400;600;700;900&display=swap');
*{margin:0;padding:0;box-sizing:border-box;-webkit-tap-highlight-color:transparent;}
:root{--st:env(safe-area-inset-top,0px);--sb:env(safe-area-inset-bottom,0px);}
html,body{height:100%;background:#0a0a1a;color:#fff;font-family:'Inter','Noto Sans JP',sans-serif;}
body{display:flex;flex-direction:column;overflow:hidden;}

/* Download bar: visible only inside Claude preview, hidden on real sites */
#dl-bar{background:#1a180a;border-bottom:2px solid #ffd700;padding:9px 20px;display:flex;align-items:center;justify-content:space-between;flex-shrink:0;gap:12px;}
#dl-bar span{font-size:12px;color:#ffd700;font-weight:600;}
#dl-bar button{background:linear-gradient(135deg,#c8102e,#ff3355);color:#fff;border:none;border-radius:8px;padding:8px 18px;font-size:12px;font-weight:700;cursor:pointer;white-space:nowrap;font-family:inherit;}

.hdr{background:linear-gradient(135deg,#c8102e,#8b0000);padding:calc(var(--st)+10px) 20px 10px;display:flex;align-items:center;justify-content:space-between;flex-shrink:0;}
.logo{font-size:22px;font-weight:900;letter-spacing:2px;}.logo b{color:#ffd700;}
.hdr-r{font-size:12px;opacity:.75;text-align:right;}
.pg{display:none;flex:1;overflow-y:auto;-webkit-overflow-scrolling:touch;padding:20px 20px calc(var(--sb)+20px);}
.pg.on{display:block;}
#pg-role.on{display:flex;flex-direction:column;align-items:center;justify-content:center;text-align:center;gap:20px;min-height:calc(100vh - 80px);}
.rcards{display:flex;gap:16px;flex-wrap:wrap;justify-content:center;max-width:560px;width:100%;}
.rcard{flex:1;min-width:200px;background:#1a1a2e;border:2px solid #333;border-radius:20px;padding:28px 18px;cursor:pointer;transition:all .2s;}
.rcard:hover,.rcard:active{border-color:#ffd700;transform:scale(1.03);}
.rcard-icon{font-size:46px;margin-bottom:12px;}
.rcard-name{font-size:20px;font-weight:700;margin-bottom:6px;}
.rcard-desc{font-size:12px;opacity:.5;line-height:1.6;}
.card{background:#1a1a2e;border-radius:16px;padding:18px;margin-bottom:14px;}
.g2{display:grid;grid-template-columns:1fr 1fr;gap:16px;max-width:960px;}
@media(max-width:640px){.g2{grid-template-columns:1fr;}}
.lb-row{display:flex;align-items:center;background:#1a1a2e;border-radius:13px;padding:14px 18px;margin-bottom:10px;border:2px solid #2a2a4a;}
.lb-row.r1{border-color:#ffd700;background:#1a180a;}.lb-row.r2{border-color:#c0c0c0;background:#141418;}.lb-row.r3{border-color:#cd7f32;background:#181208;}
.lb-rank{font-size:26px;font-weight:900;width:50px;}.lb-emoji{font-size:26px;margin:0 12px;}.lb-name{flex:1;font-size:17px;font-weight:700;}
.lb-score{font-size:22px;font-weight:900;color:#00ff88;}
.stat-box{text-align:center;}.stat-num{font-size:36px;font-weight:900;}.stat-lbl{font-size:12px;opacity:.5;margin-top:2px;}
.dot{width:8px;height:8px;border-radius:50%;background:#00ff88;display:inline-block;animation:bl 1.5s infinite;}
@keyframes bl{0%,100%{opacity:1;}50%{opacity:.2;}}
input[type=text]{width:100%;background:#0d0d20;border:2px solid #333;border-radius:10px;padding:13px 16px;font-size:17px;color:#fff;font-family:inherit;outline:none;}
input[type=text]:focus{border-color:#ffd700;}
.btn-main{width:100%;padding:16px;border:none;border-radius:13px;font-size:17px;font-weight:700;cursor:pointer;background:linear-gradient(135deg,#c8102e,#ff3355);color:#fff;font-family:inherit;margin-top:10px;}
.btn-main:active{transform:scale(.98);}
.btn-main:disabled{opacity:.5;cursor:not-allowed;transform:none;}
.btn-next{width:100%;padding:14px;border:none;border-radius:13px;font-size:16px;font-weight:700;cursor:pointer;background:linear-gradient(135deg,#1a5a3a,#00aa55);color:#fff;font-family:inherit;margin-top:8px;}
.q-badge{display:inline-block;background:#c8102e;padding:5px 14px;border-radius:20px;font-size:12px;font-weight:600;}
.pbar-wrap{background:#1a1a2e;border-radius:6px;height:8px;overflow:hidden;margin:10px 0 16px;}
.pbar{height:100%;background:linear-gradient(90deg,#c8102e,#ffd700);border-radius:6px;transition:width .4s;}
.q-en{font-size:20px;font-weight:700;line-height:1.35;margin-bottom:6px;}
.q-jp{font-size:13px;opacity:.55;margin-bottom:18px;line-height:1.5;}
.opt{width:100%;background:#1a1a2e;border:2px solid #2a2a4a;border-radius:13px;padding:15px 16px;font-size:15px;font-weight:600;color:#fff;cursor:pointer;text-align:left;margin-bottom:10px;font-family:inherit;display:block;transition:all .2s;}
.opt:hover{border-color:#555;}
.opt.correct{border-color:#00ff88!important;background:#062016!important;color:#00ff88;}
.opt.wrong{border-color:#ff3355!important;background:#200610!important;opacity:.6;}
.opt-l{color:#ffd700;font-weight:700;margin-right:8px;}
.opt-jp{font-size:11px;opacity:.5;margin-top:3px;}
.exp-box{background:#062016;border:2px solid #00aa55;border-radius:12px;padding:14px;margin:12px 0 8px;font-size:13px;line-height:1.6;}
.exp-jp{font-size:12px;opacity:.6;margin-top:5px;}
.center-screen{display:flex;flex-direction:column;align-items:center;justify-content:center;min-height:65vh;text-align:center;gap:14px;}
.big-icon{font-size:68px;animation:fl 2s ease-in-out infinite;}
@keyframes fl{0%,100%{transform:translateY(0);}50%{transform:translateY(-10px);}}
.final-score{font-size:52px;font-weight:900;color:#ffd700;}
.cf{position:fixed;top:0;left:0;width:100%;height:100%;pointer-events:none;z-index:999;}
.hidden{display:none!important;}
.err{color:#ff6b6b;font-size:13px;margin-top:8px;padding:8px 12px;background:#200610;border-radius:8px;}
.spin{display:inline-block;width:14px;height:14px;border:2px solid #ffffff44;border-top-color:#fff;border-radius:50%;animation:spin .7s linear infinite;vertical-align:middle;margin-right:6px;}
@keyframes spin{to{transform:rotate(360deg);}}
.qr-container{background:#fff;padding:14px;border-radius:16px;display:inline-block;}
.group-grid{display:grid;grid-template-columns:repeat(4,1fr);gap:10px;margin-top:8px;}
.grp-btn{background:#1a1a2e;border:2px solid #2a2a4a;border-radius:12px;padding:14px 8px;font-size:15px;font-weight:700;color:#fff;cursor:pointer;font-family:inherit;transition:all .15s;text-align:center;}
.grp-btn:hover,.grp-btn:active,.grp-btn.sel{border-color:#ffd700;background:#1a180a;color:#ffd700;}
.prog-fill{height:6px;border-radius:3px;background:linear-gradient(90deg,#c8102e,#ffd700);transition:width .5s;}
.prog-bg{flex:1;background:#2a2a4a;border-radius:3px;overflow:hidden;}
.status-badge{display:inline-flex;align-items:center;gap:6px;background:#0d0d20;border-radius:8px;padding:6px 12px;font-size:12px;margin-top:8px;}
</style>
</head>
<body id="artifacts-component-root-html">

<!-- Download bar: only shown inside Claude (window.claude exists), hidden on GitHub Pages -->
<div id="dl-bar" style="display: flex;">
  <span>⬇ Download to deploy on GitHub Pages</span>
  <button id="dl-btn" onclick="dlFile()">Download HTML</button>
</div>

<div class="hdr">
  <div class="logo">ELJ<b>2030</b></div>
  <div class="hdr-r" id="hdr-r">Japan Leadership Conference</div>
</div>

<!-- ROLE PICKER -->
<div class="pg on" id="pg-role">
  <div style="font-size:26px;font-weight:900;margin-bottom:6px;">ELJ2030 Quiz</div>
  <div style="font-size:14px;opacity:.45;margin-bottom:24px;">Japan Leadership Conference</div>
  <div class="rcards">
    <div class="rcard" onclick="goHost()">
      <div class="rcard-icon">🖥️</div>
      <div class="rcard-name">Host</div>
      <div class="rcard-desc">Show QR code &amp; live leaderboard</div>
    </div>
    <div class="rcard" onclick="goParticipant()">
      <div class="rcard-icon">📱</div>
      <div class="rcard-name">Participant</div>
      <div class="rcard-desc">Scan QR or tap here to join</div>
    </div>
  </div>
</div>

<!-- HOST -->
<div class="pg" id="pg-host">
  <div style="max-width:960px;">
    <div style="font-size:24px;font-weight:900;margin-bottom:4px;">🖥️ Host Dashboard</div>
    <div style="font-size:13px;margin-bottom:4px;"><span class="dot"></span> <span style="color:#00ff88;">Live — participants scan QR to join</span></div>
    <div id="sync-status" class="status-badge" style="margin-bottom:16px;">⏳ Connecting…</div>
    <div class="g2">
      <div>
        <div class="card" style="text-align:center;">
          <div style="font-size:11px;opacity:.5;margin-bottom:12px;text-transform:uppercase;letter-spacing:.5px;">📱 Scan to join — any iPhone</div>
          <div id="qr-wrap" style="display:flex;align-items:center;justify-content:center;min-height:240px;">
            <div style="opacity:.4;font-size:13px;">Generating QR…</div>
          </div>
          <div id="qr-url-display" style="font-size:10px;color:#ffd700;margin-top:10px;word-break:break-all;padding:0 8px;opacity:.7;"></div>
        </div>
        <div class="card">
          <div style="display:grid;grid-template-columns:1fr 1fr 1fr;gap:8px;">
            <div class="stat-box"><div class="stat-num" style="color:#00ff88;" id="h-done">0</div><div class="stat-lbl">Finished</div></div>
            <div class="stat-box"><div class="stat-num" style="color:#ffd700;" id="h-prog">0</div><div class="stat-lbl">In Progress</div></div>
            <div class="stat-box"><div class="stat-num" style="color:#4fc3f7;" id="h-avg">—</div><div class="stat-lbl">Avg Score</div></div>
          </div>
        </div>
        <div class="card">
          <div style="font-size:12px;opacity:.5;margin-bottom:10px;">Per-question accuracy</div>
          <div id="h-qstats" style="display:flex;flex-wrap:wrap;gap:6px;"></div>
        </div>
        <button onclick="hostReset()" style="padding:10px 20px;border:none;border-radius:10px;background:#2a2a4a;color:#fff;font-family:inherit;font-size:13px;font-weight:600;cursor:pointer;margin-top:4px;">🔄 Reset All</button>
      </div>
      <div>
        <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:12px;">
          <div style="font-size:18px;font-weight:700;">🏆 Leaderboard</div>
          <div id="lb-count" style="font-size:13px;opacity:.5;"></div>
        </div>
        <div id="lb-list"><div style="text-align:center;padding:30px;opacity:.4;font-size:14px;">Waiting for participants…<br>参加者を待っています</div></div>
      </div>
    </div>
  </div>
</div>

<!-- PARTICIPANT JOIN -->
<div class="pg" id="pg-join">
  <div style="text-align:center;padding-top:8px;margin-bottom:24px;">
    <div style="font-size:24px;font-weight:900;">Welcome! ようこそ</div>
    <div style="font-size:13px;opacity:.45;margin-top:4px;">ELJ2030 Quiz · 15 questions</div>
  </div>
  <div class="card">
    <div style="font-size:13px;font-weight:700;margin-bottom:12px;">Select your group number<br><span style="font-size:11px;opacity:.5;font-weight:400;">グループ番号を選んでください</span></div>
    <div class="group-grid" id="group-grid"></div>
  </div>
  <div id="join-err" class="err hidden"></div>
  <button class="btn-main" id="start-btn" onclick="startQuiz()">🚀 Start Quiz / クイズを始める</button>
</div>

<!-- QUESTION -->
<div class="pg" id="pg-question">
  <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:4px;">
    <div class="q-badge" id="q-cat"></div>
    <div style="font-size:12px;opacity:.5;" id="q-prog"></div>
  </div>
  <div class="pbar-wrap"><div class="pbar" id="q-pbar" style="width:0%"></div></div>
  <div class="q-en" id="q-en"></div>
  <div class="q-jp" id="q-jp"></div>
  <div id="q-opts"></div>
  <div class="exp-box hidden" id="q-exp">
    <div style="color:#00ff88;font-weight:700;margin-bottom:6px;" id="q-exp-ans"></div>
    <div id="q-exp-en"></div>
    <div class="exp-jp" id="q-exp-jp"></div>
  </div>
  <button class="btn-next hidden" id="q-next" onclick="nextQ()">Next ⏭</button>
</div>

<!-- FINAL -->
<div class="pg" id="pg-final">
  <div class="center-screen">
    <div style="font-size:26px;font-weight:900;">🎉 Quiz Complete!</div>
    <div style="font-size:13px;opacity:.5;">お疲れ様でした！</div>
    <div class="big-icon" id="f-icon"></div>
    <div style="font-size:17px;font-weight:700;" id="f-name"></div>
    <div class="final-score" id="f-score"></div>
    <div style="font-size:13px;opacity:.55;" id="f-rank"></div>
    <div style="background:#1a1a2e;border-radius:14px;padding:16px;width:100%;max-width:380px;">
      <div style="font-size:12px;font-weight:700;opacity:.6;margin-bottom:10px;text-transform:uppercase;letter-spacing:.5px;">Your answers</div>
      <div id="f-breakdown"></div>
    </div>
    <div style="font-size:13px;opacity:.45;text-align:center;line-height:1.7;">Thank you for participating!<br>ご参加ありがとうございました！</div>
  </div>
  <div class="cf" id="cf"></div>
</div>

<script>
const QS=[
  {cat:"Customer Excellence",cjp:"顧客エクセレンス",en:"What does a customer-centric approach in ELJ2030 primarily focus on?",jp:"ELJ2030における顧客中心のアプローチが主に焦点を当てるのは？",opts:["Reducing costs","Understanding and anticipating patient/customer needs","Increasing sales targets","Streamlining internal processes"],ojp:["コスト削減","患者・顧客のニーズを理解し先取りすること","販売目標の引き上げ","社内プロセスの効率化"],ans:1,exen:"ELJ2030 centers on deeply understanding patient and customer needs, placing them at the heart of every decision.",exjp:"ELJ2030は患者・顧客のニーズを深く理解し、意思決定の中心に置くことを重視します。"},
  {cat:"Customer Excellence",cjp:"顧客エクセレンス",en:"Which behavior best demonstrates Customer Excellence at ELJ?",jp:"ELJにおいて顧客エクセレンスを最もよく示す行動はどれ？",opts:["Following standard scripts","Co-creating solutions with healthcare professionals","Minimising time with customers","Focusing only on promoted products"],ojp:["標準スクリプトを使用する","医療従事者と共にソリューションを共創する","顧客との接触時間を最小化する","推奨製品のみに集中する"],ans:1,exen:"Customer Excellence means co-creating value with healthcare professionals beyond transactions.",exjp:"顧客エクセレンスとは医療従事者と共に価値を共創することです。"},
  {cat:"Customer Excellence",cjp:"顧客エクセレンス",en:"In ELJ2030, customer excellence includes which of the following?",jp:"ELJ2030において顧客エクセレンスに含まれるのは？",opts:["One-size-fits-all messaging","Personalised, insight-driven engagement","Reducing field force size","Focusing on tier-1 hospitals only"],ojp:["画一的なメッセージング","個別化されたインサイト主導のエンゲージメント","MRの人数削減","一次病院のみへの集中"],ans:1,exen:"Personalised, insight-driven engagement makes every interaction relevant and impactful.",exjp:"個別化されたエンゲージメントにより各顧客との接点が効果的になります。"},
  {cat:"Digital & Data",cjp:"デジタル＆データ",en:"How does ELJ2030 leverage digital and data capabilities?",jp:"ELJ2030はデジタル・データをどのように活用するか？",opts:["To replace human judgment entirely","To generate insights that improve decisions","Only for back-office automation","To reduce headcount"],ojp:["人間の判断を完全に置き換えるため","意思決定を向上させる洞察を生み出すため","バックオフィスの自動化のみ","人員削減のため"],ans:1,exen:"Digital and data tools generate actionable insights that enhance human judgment in the field.",exjp:"デジタル・データツールは現場の人間の判断を強化する洞察を提供します。"},
  {cat:"Digital & Data",cjp:"デジタル＆データ",en:"What is a key outcome of building strong data literacy in ELJ?",jp:"ELJでデータリテラシーを高める重要な成果は？",opts:["Eliminating sales calls","Empowering evidence-based decisions","Automating all interactions","Reducing training budgets"],ojp:["営業訪問をなくすこと","エビデンスに基づく意思決定を支援すること","すべての対応を自動化すること","トレーニング予算の削減"],ans:1,exen:"Data literacy empowers every employee to interpret data confidently for better decisions.",exjp:"データリテラシーにより従業員がデータを自信を持って解釈できます。"},
  {cat:"Digital & Data",cjp:"デジタル＆データ",en:"Which best describes the role of AI in ELJ2030?",jp:"ELJ2030におけるAIの役割を最もよく表すのは？",opts:["Fully autonomous decisions","An enabler of smarter, human-led strategies","A replacement for field teams","Only used in R&D"],ojp:["完全自律型の意思決定","より賢い人間主導の戦略を可能にするもの","現場チームの代替","R&Dのみで使用"],ans:1,exen:"AI in ELJ2030 augments human capabilities while keeping people in the lead.",exjp:"ELJ2030のAIは人間の能力を拡張します。"},
  {cat:"Innovation Agility",cjp:"イノベーション・アジリティ",en:"What does Innovation Agility mean in the ELJ2030 context?",jp:"ELJ2030のイノベーション・アジリティとは？",opts:["Avoiding change","Rapidly testing, learning, and adapting","Only innovating in product development","Delegating innovation to HQ"],ojp:["変化を避けること","新しいアイデアを迅速にテスト・学習・適応すること","製品開発のみでイノベーション","イノベーションを本社に委任すること"],ans:1,exen:"Innovation Agility means rapid experimentation — test, learn, fail fast, and scale what works.",exjp:"イノベーション・アジリティとは迅速な実験です。"},
  {cat:"Innovation Agility",cjp:"イノベーション・アジリティ",en:"Which mindset best supports Innovation Agility?",jp:"イノベーション・アジリティを最もよく支えるマインドセットは？",opts:["'If it ain't broke, don't fix it'","Psychological safety to experiment and learn","Strict hierarchy for decision-making","Annual planning cycles only"],ojp:["「壊れていないなら直すな」","実験し失敗から学ぶ心理的安全性","意思決定のための厳格な階層","年次計画サイクルのみ"],ans:1,exen:"Psychological safety enables bold experimentation and learning from setbacks.",exjp:"心理的安全性により大胆な実験と失敗からの学びが可能です。"},
  {cat:"Innovation Agility",cjp:"イノベーション・アジリティ",en:"How should teams approach new market opportunities in ELJ2030?",jp:"ELJ2030において新市場機会へのアプローチは？",opts:["Wait for proven global models","Pilot locally, learn quickly, scale successes","Focus only on existing products","Seek approval for every small action"],ojp:["実証済みのグローバルモデルを待つ","ローカルでパイロット、迅速に学習、成功をスケール","既存製品のみに集中","小さなアクションごとに承認を求める"],ans:1,exen:"Pilot-learn-scale: start small, gather insights fast, and scale what delivers results.",exjp:"「パイロット・学習・スケール」アプローチを実践します。"},
  {cat:"People & Culture",cjp:"ピープル＆カルチャー",en:"What is a core element of the People & Culture capability in ELJ2030?",jp:"ELJ2030のピープル＆カルチャーの中核要素は？",opts:["Maximising individual performance only","Building an inclusive, growth-oriented culture","Minimising training investment","Standardising all roles globally"],ojp:["個人パフォーマンスの最大化のみ","包括的で成長志向の文化を構築すること","トレーニング投資の最小化","すべての役割をグローバルに標準化"],ans:1,exen:"People & Culture emphasises an inclusive environment where everyone can grow and thrive.",exjp:"誰もが成長できる包括的な環境の構築を重視します。"},
  {cat:"People & Culture",cjp:"ピープル＆カルチャー",en:"How does ELJ2030 support employee development?",jp:"ELJ2030はどのように従業員の成長を支援するか？",opts:["One-time onboarding training only","Continuous learning, coaching and career growth","Focusing only on top performers","External hiring over internal development"],ojp:["入社時研修のみ","継続的な学習、コーチング、キャリア成長機会","ハイパフォーマーのみへの集中","内部育成より外部採用"],ans:1,exen:"ELJ2030 invests in continuous development — coaching, pathways, and opportunities for every employee.",exjp:"ELJ2030はすべての従業員の継続的な成長に投資します。"},
  {cat:"People & Culture",cjp:"ピープル＆カルチャー",en:"Diversity and inclusion in ELJ2030 is seen as:",jp:"ELJ2030におけるD&Iは何と見なされているか？",opts:["A compliance requirement","A strategic advantage that drives innovation","Primarily an HR concern","Nice-to-have, not essential"],ojp:["コンプライアンス要件","イノベーションを促進する戦略的優位性","主にHRの関心事","必須ではなくあれば良いもの"],ans:1,exen:"D&I is a strategic driver — diverse perspectives fuel innovation and create stronger teams.",exjp:"D&Iは戦略的推進力であり多様な視点がイノベーションを促します。"},
  {cat:"Cross-functional Collaboration",cjp:"クロスファンクショナル協働",en:"Why is cross-functional collaboration critical for ELJ2030?",jp:"クロスファンクショナル協働がELJ2030にとって重要な理由は？",opts:["It reduces individual accountability","Complex challenges require diverse expertise","It allows teams to avoid decisions","It is required by global HQ only"],ojp:["個人の説明責任を軽減するから","複雑な課題には多様な専門知識の協働が必要だから","チームが意思決定を避けられるから","グローバル本社の要件だから"],ans:1,exen:"Complex challenges require diverse expertise working together to create breakthrough solutions.",exjp:"複雑な課題はクロスファンクショナル協働が解決します。"},
  {cat:"Cross-functional Collaboration",cjp:"クロスファンクショナル協働",en:"What enables effective cross-functional collaboration at ELJ?",jp:"ELJで効果的なクロスファンクショナル協働を可能にするものは？",opts:["Boundaries that limit sharing","Shared goals, mutual respect, and open communication","Reporting to the same line manager","Working in the same physical location"],ojp:["情報共有を制限する明確な境界","共通の目標、相互尊重、オープンなコミュニケーション","同じ直属の上司への報告","同じ物理的場所での勤務"],ans:1,exen:"Shared goals, mutual respect, and open communication enable effective collaboration.",exjp:"共通の目標と相互尊重が協働を可能にします。"},
  {cat:"Cross-functional Collaboration",cjp:"クロスファンクショナル協働",en:"In ELJ2030, breaking down silos primarily leads to:",jp:"ELJ2030においてサイロを解消することが主にもたらすものは？",opts:["Slower decision-making","Faster, innovative and customer-focused outcomes","Reduced individual ownership","More complex reporting structures"],ojp:["意思決定の遅延","より迅速で革新的な顧客中心の成果","個人のオーナーシップの低下","より複雑な報告体制"],ans:1,exen:"Breaking down silos enables faster decisions and more customer-focused solutions.",exjp:"サイロを解消することでより迅速な意思決定が実現します。"}
];

const EMOJIS=['🔵','🔴','🟡','🟢','🟠','🟣','💎','⭐','🌟','🚀','🦁','🐯','🦊','🦅','🏆','⚡','🔥','🎯','🌊','🎪'];
const JSONBIN_BASE='https://api.jsonbin.io/v3/b';
let binId=null, myId='', myGroup='', myEmoji='', qIdx=0, myAnswers=[], pollInt=null;

// ── Show download bar only inside Claude (not on GitHub Pages) ─────────────
window.addEventListener('load', ()=>{
  // window.claude exists only inside Claude's artifact sandbox
  if (typeof window.claude !== 'undefined' || window.location.href.includes('claude.ai')) {
    document.getElementById('dl-bar').style.display='flex';
  }
  const params=new URLSearchParams(window.location.search);
  if (params.get('p')==='1') {
    const bid=params.get('bid');
    if (bid) { binId=bid; try{localStorage.setItem('elj_bin',bid);}catch(e){} }
    goParticipant();
  }
});

// ── Storage (JSONBin public API — no account needed) ──────────────────────
function setSyncStatus(msg, ok){
  const el=document.getElementById('sync-status');
  if(!el) return;
  el.textContent=msg;
  el.style.color=ok?'#00ff88':'#ffd700';
}

async function initBin(){
  // Try localStorage first
  if(!binId){ try{binId=localStorage.getItem('elj_bin')||null;}catch(e){} }
  if(binId){ setSyncStatus('✅ Session ready',true); return binId; }
  try {
    setSyncStatus('⏳ Creating session…', false);
    const r=await fetch(JSONBIN_BASE,{
      method:'POST',
      headers:{'Content-Type':'application/json','X-Bin-Private':'false','X-Bin-Name':'ELJ2030'},
      body:JSON.stringify({participants:{}})
    });
    if(!r.ok) throw new Error('HTTP '+r.status);
    const d=await r.json();
    binId=d.metadata?.id||null;
    if(!binId) throw new Error('No ID returned');
    try{localStorage.setItem('elj_bin',binId);}catch(e){}
    setSyncStatus('✅ Session ready',true);
    return binId;
  } catch(e){
    setSyncStatus('⚠️ Sync error: '+e.message, false);
    return null;
  }
}

async function getSession(){
  if(!binId) return {participants:{}};
  try{
    const r=await fetch(JSONBIN_BASE+'/'+binId+'/latest');
    if(!r.ok) throw new Error('HTTP '+r.status);
    const d=await r.json();
    return d.record||{participants:{}};
  }catch(e){ return {participants:{}}; }
}

async function setSession(data){
  if(!binId) return;
  try{
    const r=await fetch(JSONBIN_BASE+'/'+binId,{
      method:'PUT',
      headers:{'Content-Type':'application/json'},
      body:JSON.stringify(data)
    });
    if(!r.ok) throw new Error('HTTP '+r.status);
  }catch(e){ console.error('setSession:',e); }
}

// ── UI ─────────────────────────────────────────────────────────────────────
function show(id){document.querySelectorAll('.pg').forEach(p=>p.classList.remove('on'));document.getElementById(id).classList.add('on');}
function showErr(id,msg){const e=document.getElementById(id);e.textContent=msg;msg?e.classList.remove('hidden'):e.classList.add('hidden');}

// ── HOST ──────────────────────────────────────────────────────────────────
async function goHost(){
  document.getElementById('hdr-r').textContent='🖥️ Host';
  show('pg-host');
  const id=await initBin();
  if(!id){ setSyncStatus('❌ Could not create session. Check internet.',false); return; }
  const base=window.location.href.split('?')[0];
  const url=base+'?p=1&bid='+encodeURIComponent(id);
  buildQR(url);
  clearInterval(pollInt);
  pollInt=setInterval(refreshLB,2000);
  refreshLB();
}

function buildQR(url){
  const wrap=document.getElementById('qr-wrap');
  wrap.innerHTML='';
  document.getElementById('qr-url-display').textContent=url;
  try{
    const c=document.createElement('div');
    c.className='qr-container';
    wrap.appendChild(c);
    new QRCode(c,{text:url,width:220,height:220,colorDark:'#000000',colorLight:'#ffffff',correctLevel:QRCode.CorrectLevel.M});
  }catch(e){
    wrap.innerHTML=`<div style="padding:16px;font-size:11px;word-break:break-all;opacity:.6;line-height:1.7;">${url}</div>`;
  }
}

async function refreshLB(){
  const data=await getSession();
  renderLBData(data.participants||{});
}

function renderLBData(participants){
  const parts=Object.values(participants);
  const done=parts.filter(p=>p.done);
  document.getElementById('h-done').textContent=done.length;
  document.getElementById('h-prog').textContent=parts.filter(p=>!p.done).length;
  if(done.length){
    document.getElementById('h-avg').textContent=Math.round(done.reduce((s,p)=>s+p.score,0)/done.length);
    document.getElementById('h-qstats').innerHTML=QS.map((_,i)=>{
      const c=done.filter(p=>Number(p.answers[i])===QS[i].ans).length;
      const pct=Math.round(c/done.length*100);
      const col=pct>=70?'#00ff88':pct>=40?'#ffd700':'#ff3355';
      return `<div style="background:#0d0d20;border-radius:6px;padding:5px 8px;font-size:11px;color:${col};font-weight:700;">Q${i+1} ${pct}%</div>`;
    }).join('');
  } else {
    document.getElementById('h-avg').textContent='—';
    document.getElementById('h-qstats').innerHTML='';
  }
  const allActive=[...parts].sort((a,b)=>a.done===b.done?b.score-a.score:a.done?-1:1);
  document.getElementById('lb-count').textContent=allActive.length?allActive.length+' participant'+(allActive.length>1?'s':''):'';
  const list=document.getElementById('lb-list');
  if(!allActive.length){list.innerHTML='<div style="text-align:center;padding:30px;opacity:.4;font-size:14px;">Waiting for participants…<br>参加者を待っています</div>';return;}
  const doneRanked=[...done].sort((a,b)=>b.score-a.score);
  list.innerHTML=allActive.map(p=>{
    const ri=doneRanked.findIndex(d=>d.id===p.id);
    const rankLabel=p.done?(ri===0?'🥇':ri===1?'🥈':ri===2?'🥉':'#'+(ri+1)):'⏳';
    const cls=p.done?(ri===0?'r1':ri===1?'r2':ri===2?'r3':''):'';
    const correct=QS.filter((_,qi)=>Number(p.answers[qi])===QS[qi].ans).length;
    const progress=p.done?QS.length:(p.qProgress||0);
    const pct=Math.round(progress/QS.length*100);
    return `<div class="lb-row ${cls}">
      <div class="lb-rank">${rankLabel}</div>
      <div class="lb-emoji">${p.emoji||'🔵'}</div>
      <div style="flex:1;">
        <div class="lb-name">${p.name}</div>
        <div style="font-size:11px;color:#ffd700;margin-top:2px;">${p.done?correct+'/'+QS.length+' correct':'Q'+(progress+1)+'/'+QS.length+' in progress'}</div>
        ${!p.done?`<div style="display:flex;align-items:center;gap:6px;margin-top:5px;"><div class="prog-bg"><div class="prog-fill" style="width:${pct}%"></div></div><span style="font-size:11px;opacity:.4;">${pct}%</span></div>`:''}
      </div>
      <div style="text-align:right;"><div class="lb-score">${p.score}</div><div style="font-size:12px;opacity:.5;">pts</div></div>
    </div>`;
  }).join('');
}

async function hostReset(){
  if(!confirm('Reset all data?\n全データをリセットしますか？')) return;
  try{localStorage.removeItem('elj_bin');}catch(e){}
  binId=null;
  await initBin();
  renderLBData({});
  ['h-done','h-prog'].forEach(id=>document.getElementById(id).textContent='0');
  document.getElementById('h-avg').textContent='—';
  document.getElementById('h-qstats').innerHTML='';
  document.getElementById('lb-list').innerHTML='<div style="text-align:center;padding:30px;opacity:.4;font-size:14px;">Waiting for participants…<br>参加者を待っています</div>';
  // Rebuild QR with new binId
  const base=window.location.href.split('?')[0];
  buildQR(base+'?p=1&bid='+encodeURIComponent(binId||''));
}

// ── PARTICIPANT ───────────────────────────────────────────────────────────
function goParticipant(){
  document.getElementById('hdr-r').textContent='📱 Participant';
  const grid=document.getElementById('group-grid');
  grid.innerHTML='';
  for(let i=1;i<=20;i++){
    const b=document.createElement('button');
    b.className='grp-btn';b.textContent='Group '+i;
    b.onclick=()=>{document.querySelectorAll('.grp-btn').forEach(x=>x.classList.remove('sel'));b.classList.add('sel');myGroup='Group '+i;showErr('join-err','');};
    grid.appendChild(b);
  }
  show('pg-join');
}

async function startQuiz(){
  showErr('join-err','');
  if(!myGroup){showErr('join-err','Please select your group / グループを選んでください');return;}
  if(!binId){showErr('join-err','No session found. Please re-scan the QR code.');return;}
  const btn=document.getElementById('start-btn');
  btn.disabled=true;btn.innerHTML='<span class="spin"></span>Starting…';
  myId='p_'+Date.now()+'_'+Math.random().toString(36).substr(2,5);
  myEmoji=EMOJIS[Math.floor(Math.random()*EMOJIS.length)];
  qIdx=0;myAnswers=[];
  const data=await getSession();
  if(!data.participants) data.participants={};
  data.participants[myId]={id:myId,name:myGroup,emoji:myEmoji,score:0,done:false,qProgress:0,answers:{}};
  await setSession(data);
  btn.disabled=false;btn.innerHTML='🚀 Start Quiz / クイズを始める';
  show('pg-question');renderQ();
}

function renderQ(){
  const q=QS[qIdx];
  document.getElementById('q-cat').textContent=q.cat+' / '+q.cjp;
  document.getElementById('q-prog').textContent=(qIdx+1)+' / '+QS.length;
  document.getElementById('q-pbar').style.width=(qIdx/QS.length*100)+'%';
  document.getElementById('q-en').textContent=q.en;
  document.getElementById('q-jp').textContent=q.jp;
  document.getElementById('q-exp').classList.add('hidden');
  document.getElementById('q-next').classList.add('hidden');
  const c=document.getElementById('q-opts');c.innerHTML='';
  q.opts.forEach((o,i)=>{
    const b=document.createElement('button');b.className='opt';
    b.innerHTML=`<span class="opt-l">${String.fromCharCode(65+i)}.</span>${o}<div class="opt-jp">${q.ojp[i]}</div>`;
    b.onclick=()=>pickAns(i);c.appendChild(b);
  });
}

async function pickAns(ai){
  document.querySelectorAll('.opt').forEach(b=>{b.onclick=null;b.style.cursor='default';});
  const q=QS[qIdx];
  document.querySelectorAll('.opt').forEach((b,i)=>{
    if(i===q.ans) b.classList.add('correct');
    else if(i===ai&&ai!==q.ans) b.classList.add('wrong');
  });
  myAnswers[qIdx]=ai;
  const score=myAnswers.reduce((s,a,i)=>s+(a!==undefined&&a===QS[i].ans?10:0),0);
  const answersObj={};myAnswers.forEach((a,i)=>answersObj[i]=a);
  const data=await getSession();
  if(data.participants?.[myId]){
    data.participants[myId].score=score;
    data.participants[myId].qProgress=qIdx+1;
    data.participants[myId].answers=answersObj;
  }
  await setSession(data);
  document.getElementById('q-exp-ans').textContent='✅ '+String.fromCharCode(65+q.ans)+'. '+q.opts[q.ans];
  document.getElementById('q-exp-en').textContent=q.exen;
  document.getElementById('q-exp-jp').textContent=q.exjp;
  document.getElementById('q-exp').classList.remove('hidden');
  const nb=document.getElementById('q-next');
  nb.textContent=qIdx<QS.length-1?'Next Question ⏭':'See My Results 🏆';
  nb.classList.remove('hidden');
}

function nextQ(){qIdx++;if(qIdx>=QS.length){finish();return;}renderQ();}

async function finish(){
  const score=myAnswers.reduce((s,a,i)=>s+(a===QS[i].ans?10:0),0);
  const answersObj={};myAnswers.forEach((a,i)=>answersObj[i]=a);
  const data=await getSession();
  if(!data.participants) data.participants={};
  data.participants[myId]={id:myId,name:myGroup,emoji:myEmoji,score,done:true,qProgress:QS.length,answers:answersObj};
  await setSession(data);
  const done=Object.values(data.participants).filter(p=>p.done).sort((a,b)=>b.score-a.score);
  const rank=done.findIndex(p=>p.id===myId)+1;
  document.getElementById('f-icon').textContent=rank===1?'🥇':rank===2?'🥈':rank===3?'🥉':'#'+rank;
  document.getElementById('f-name').textContent=myEmoji+' '+myGroup;
  document.getElementById('f-score').textContent=score+' pts';
  document.getElementById('f-rank').textContent='Rank '+rank+' of '+done.length+' completed';
  document.getElementById('f-breakdown').innerHTML=QS.map((q,i)=>{
    const ok=myAnswers[i]===q.ans;
    return `<div style="display:flex;align-items:center;gap:8px;margin-bottom:6px;font-size:13px;">
      <span>${ok?'✅':'❌'}</span><span style="flex:1;opacity:.6;">${q.cat}</span>
      <span style="font-weight:700;color:${ok?'#00ff88':'#ff6b6b'}">${ok?'+10':'0'}</span></div>`;
  }).join('');
  document.getElementById('hdr-r').textContent=myEmoji+' '+myGroup+' — '+score+' pts';
  show('pg-final');
  if(score>=120) launchConfetti();
}

function launchConfetti(){
  const c=document.getElementById('cf');c.innerHTML='';
  const s=document.createElement('style');s.textContent='@keyframes cf{to{top:110vh;transform:rotate(720deg);}}';c.appendChild(s);
  const cols=['#ffd700','#c8102e','#00ff88','#4fc3f7','#ff80ab','#fff'];
  for(let i=0;i<120;i++){
    const d=document.createElement('div');const col=cols[i%cols.length];const sz=Math.random()*10+5;
    d.style.cssText=`position:absolute;width:${sz}px;height:${sz}px;background:${col};left:${Math.random()*100}%;top:-20px;border-radius:${Math.random()>.5?'50%':'2px'};animation:cf ${Math.random()*3+2}s linear ${Math.random()*2}s forwards;`;
    c.appendChild(d);
  }
}

function dlFile(){
  const html='<!DOCTYPE html>\n'+document.documentElement.outerHTML;
  const blob=new Blob([html],{type:'text/html'});
  const a=document.createElement('a');a.href=URL.createObjectURL(blob);a.download='index.html';
  document.body.appendChild(a);a.click();document.body.removeChild(a);
  document.getElementById('dl-btn').textContent='✅ Downloaded!';
}
</script>

</body></html>
