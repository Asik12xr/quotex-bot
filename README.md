(function () {
  'use strict';
  try {
    var BOX_ID = 'qxAutoWatchBookmarkAI';
    var TMR_KEY = '_qxAutoWatchBookmarkTMR';
    var oldBox = document.getElementById(BOX_ID);
    if (oldBox) oldBox.remove();
    if (window[TMR_KEY]) clearInterval(window[TMR_KEY]);

    // Python auto_watch_signal.py-এর মূল প্যারামিটার ও কনফ্লুয়েন্স নিয়ম
    var CANDLE_PERIOD_SEC = 60;
    var TARGET_COUNTDOWN_SEC = 3;
    var MIN_FINAL_SCORE = 1.95;
    var MIN_SCORE_GAP = 0.45;
    var MAX_SIGNALS_PER_SESSION = 60;

    var voiceEnabled = true;
    var signalsEmitted = 0;
    var lastTriggeredMinute = '';
    var currentMinuteKey = '';
    var minuteSamples = [];
    var shareVideo = null;
    var shareStream = null;
    var shareCanvas = null;

    var box = document.createElement('div');
    box.id = BOX_ID;
    box.style.cssText = 'position:fixed;top:16px;right:14px;z-index:2147483647;width:312px;background:#090e1a;color:#f8fafc;border:2px solid #38bdf8;border-radius:12px;padding:10px 12px;font:12px/1.38 sans-serif;touch-action:none;user-select:none;box-shadow:0 12px 30px rgba(0,0,0,.65)';
    box.innerHTML =
      '<div style="cursor:move;display:flex;justify-content:space-between;align-items:center;padding-bottom:5px;border-bottom:1px solid #1e293b;"><b style="color:#38bdf8;">Auto Watch Screen AI</b><span id="awCd" style="color:#facc15;font-weight:700;">00:60</span></div>' +
      '<div id="awSg" style="font-size:19px;font-weight:800;color:#facc15;margin:6px 0 3px;">WAIT</div>' +
      '<div id="awRs" style="color:#e2e8f0;margin-bottom:4px;">স্ক্রিন শেয়ার বা চার্ট স্ক্যান চলছে...</div>' +
      '<div id="awSc" style="font-size:11px;color:#94a3b8;margin-bottom:8px;">1m Mem: 0 | UP: 0.00 | DOWN: 0.00</div>' +
      '<div style="display:flex;gap:6px;margin-bottom:6px;"><button id="awNow" style="flex:1;padding:5px 7px;border:0;border-radius:6px;background:#0284c7;color:#fff;font-weight:700;cursor:pointer;">এখনই স্ক্যান</button><button id="awVoc" style="padding:5px 8px;border:0;border-radius:6px;background:#334155;color:#fff;font-weight:700;cursor:pointer;">ভয়েস ON</button><button id="awCls" style="padding:5px 8px;border:0;border-radius:6px;background:#991b1b;color:#fff;font-weight:700;cursor:pointer;">X</button></div>' +
      '<button id="awShr" style="width:100%;padding:6px 8px;border:0;border-radius:6px;background:#7c3aed;color:#fff;font-weight:800;cursor:pointer;">স্ক্রিন শেয়ার দিয়ে স্ক্রিন দেখো</button>';
    document.body.appendChild(box);

    var drag = 0, sx = 0, sy = 0, ox = 0, oy = 0;
    box.onpointerdown = function (e) {
      if (e.target.tagName === 'BUTTON') return;
      drag = 1;
      var r = box.getBoundingClientRect();
      box.style.left = r.left + 'px';
      box.style.top = r.top + 'px';
      box.style.right = 'auto';
      ox = r.left; oy = r.top; sx = e.clientX; sy = e.clientY;
      try { box.setPointerCapture(e.pointerId); } catch (_) {}
    };
    box.onpointermove = function (e) {
      if (!drag) return;
      var nx = ox + (e.clientX - sx), ny = oy + (e.clientY - sy);
      var mx = Math.max(0, window.innerWidth - box.offsetWidth), my = Math.max(0, window.innerHeight - box.offsetHeight);
      box.style.left = Math.min(mx, Math.max(0, nx)) + 'px';
      box.style.top = Math.min(my, Math.max(0, ny)) + 'px';
    };
    box.onpointerup = box.onpointercancel = function () { drag = 0; };

    function speakBn(text) {
      if (!voiceEnabled || !window.speechSynthesis) return;
      try {
        window.speechSynthesis.cancel();
        var u = new SpeechSynthesisUtterance(text);
        u.lang = 'bn-BD';
        u.rate = 1.0;
        window.speechSynthesis.speak(u);
      } catch (_) {}
    }

    function getCandleRemSec(nowDt) {
      var rem = CANDLE_PERIOD_SEC - nowDt.getSeconds();
      return rem <= 0 ? CANDLE_PERIOD_SEC : rem;
    }

    function formatCd(rem) {
      return '00:' + (rem < 10 ? '0' + rem : rem);
    }

    function isBullPx(r, g, b) {
      return g >= 80 && (g - r) >= 22 && (g - b) >= 10;
    }

    function isBearPx(r, g, b) {
      return r >= 80 && (r - g) >= 20 && (r - b) >= 16;
    }

    function stopShare() {
      try {
        if (shareStream && shareStream.getTracks) {
          shareStream.getTracks().forEach(function (t) { t.stop(); });
        }
      } catch (_) {}
      shareStream = null;
      var b = document.getElementById('awShr');
      if (b) {
        b.textContent = 'স্ক্রিন শেয়ার দিয়ে স্ক্রিন দেখো';
        b.style.background = '#7c3aed';
      }
    }

    function startShare() {
      var md = navigator.mediaDevices;
      if (!md || !md.getDisplayMedia) {
        document.getElementById('awRs').textContent = 'এই ব্রাউজারে স্ক্রিন শেয়ার সাপোর্ট নেই';
        speakBn('এই ব্রাউজারে স্ক্রিন শেয়ার সাপোর্ট করে না');
        return;
      }
      md.getDisplayMedia({ video: { frameRate: 5 }, audio: false }).then(function (stream) {
        shareStream = stream;
        if (!shareVideo) {
          shareVideo = document.createElement('video');
          shareVideo.muted = true;
          shareVideo.playsInline = true;
        }
        shareVideo.srcObject = stream;
        shareVideo.play().catch(function () {});
        if (!shareCanvas) shareCanvas = document.createElement('canvas');
        var b = document.getElementById('awShr');
        if (b) {
          b.textContent = 'স্ক্রিন শেয়ার চালু (বন্ধ করতে চাপো)';
          b.style.background = '#6d28d9';
        }
        speakBn('স্ক্রিন শেয়ার চালু হয়েছে, স্ক্রিন স্ক্যান করা হচ্ছে');
        tickWatcher(true);
      }).catch(function () {
        document.getElementById('awRs').textContent = 'স্ক্রিন শেয়ার অনুমতি দেওয়া হয়নি';
        speakBn('স্ক্রিন শেয়ার অনুমতি দেওয়া হয়নি');
      });
    }

    function parseSegmentsFromImageData(px, w, h, cropBox) {
      var x0 = Math.max(0, Math.floor(w * cropBox.x0));
      var y0 = Math.max(0, Math.floor(h * cropBox.y0));
      var x1 = Math.min(w, Math.ceil(w * cropBox.x1));
      var y1 = Math.min(h, Math.ceil(h * cropBox.y1));
      var cw = Math.max(10, x1 - x0);
      var ch = Math.max(10, y1 - y0);

      var numSlices = 12;
      var segs = [];
      var bullAll = 0, bearAll = 0, ysAll = [];

      for (var i = 0; i < numSlices; i++) {
        var sx0 = x0 + Math.floor(cw * (0.04 + (0.92 * i / numSlices)));
        var sx1 = x0 + Math.max(1, Math.floor(cw * (0.04 + (0.92 * (i + 1) / numSlices))));
        var bull = 0, bear = 0, ys = [];

        for (var y = y0; y < y1; y += 2) {
          for (var x = sx0; x < sx1; x += 2) {
            var idx = (y * w + x) * 4;
            if (px[idx + 3] < 10) continue;
            var r = px[idx], g = px[idx + 1], b = px[idx + 2];
            var ny = (y - y0) / ch;
            if (isBullPx(r, g, b)) {
              bull++;
              ys.push(ny);
              bullAll++;
              ysAll.push(ny);
            } else if (isBearPx(r, g, b)) {
              bear++;
              ys.push(ny);
              bearAll++;
              ysAll.push(ny);
            }
          }
        }

        if (!ys.length) {
          segs.push({ bull: bull, bear: bear, total: 0, avg_y: 0.5, top_y: 0.5, bot_y: 0.5, span_y: 0.0 });
        } else {
          var avgY = ys.reduce(function (a, v) { return a + v; }, 0) / ys.length;
          var topY = Math.min.apply(null, ys);
          var botY = Math.max.apply(null, ys);
          segs.push({
            bull: bull,
            bear: bear,
            total: bull + bear,
            avg_y: avgY,
            top_y: topY,
            bot_y: botY,
            span_y: Math.max(0.0, botY - topY)
          });
        }
      }

      if ((bullAll + bearAll) < 6 || !ysAll.length) return null;
      if ((Math.max.apply(null, ysAll) - Math.min.apply(null, ysAll)) < 0.05) return null;
      return segs;
    }

    function captureScreenOrChartSegments() {
      // ১. স্ক্রিন শেয়ার ফ্রেম থেকে স্ক্রিন স্ক্যান
      if (shareStream && shareVideo && shareVideo.readyState >= 2 && shareCanvas) {
        try {
          var vw = Math.min(shareVideo.videoWidth || 400, 420);
          var vh = Math.min(shareVideo.videoHeight || 720, 720);
          if (vw >= 40 && vh >= 40) {
            shareCanvas.width = vw;
            shareCanvas.height = vh;
            var sctx = shareCanvas.getContext('2d');
            sctx.drawImage(shareVideo, 0, 0, vw, vh);
            var spx = sctx.getImageData(0, 0, vw, vh).data;
            var sSegs = parseSegmentsFromImageData(spx, vw, vh, { x0: 0.04, y0: 0.14, x1: 0.90, y1: 0.84 });
            if (sSegs) return { segs: sSegs, mode: 'Screen-Share' };
          }
        } catch (_) {}
      }

      // ২. পেজের দৃশ্যমান চার্ট ক্যানভাস থেকে স্ক্যান
      var cvs = [].slice.call(document.querySelectorAll('canvas')).filter(function (c) {
        if (c === shareCanvas) return false;
        var r = c.getBoundingClientRect();
        var rw = Math.max(r.width || 0, c.width || 0);
        var rh = Math.max(r.height || 0, c.height || 0);
        return rw >= 90 && rh >= 70;
      });

      var bestSegs = null;
      for (var i = 0; i < cvs.length; i++) {
        try {
          var c = cvs[i], r = c.getBoundingClientRect();
          var w = Math.min(c.width || Math.round(r.width), 360);
          var h = Math.min(c.height || Math.round(r.height), 260);
          var ctx = c.getContext('2d');
          if (!ctx || w < 24 || h < 24) continue;
          var px = ctx.getImageData(0, 0, w, h).data;
          var segs = parseSegmentsFromImageData(px, w, h, { x0: 0.04, y0: 0.05, x1: 0.95, y1: 0.95 });
          if (segs && (!bestSegs || segs.length >= bestSegs.length)) bestSegs = segs;
        } catch (_) {}
      }

      if (bestSegs) return { segs: bestSegs, mode: 'Chart-Scan' };
      return null;
    }

    function computeEma(values, period) {
      if (!values.length) return [];
      var alpha = 2.0 / (period + 1.0);
      var ema = [values[0]];
      for (var i = 1; i < values.length; i++) ema.push(alpha * values[i] + (1.0 - alpha) * ema[ema.length - 1]);
      return ema;
    }

    function computeVisualRsi(prices, period) {
      if (prices.length < 3) return [50.0, 50.0];
      var gains = [], losses = [];
      for (var i = 1; i < prices.length; i++) {
        var d = prices[i] - prices[i - 1];
        gains.push(d > 0 ? d : 0.0);
        losses.push(d < 0 ? -d : 0.0);
      }
      function calc(sg, sl) {
        var ag = sg.reduce(function (a, v) { return a + v; }, 0) / Math.max(sg.length, 1);
        var al = sl.reduce(function (a, v) { return a + v; }, 0) / Math.max(sl.length, 1);
        if (al === 0 && ag === 0) return 50.0;
        if (al === 0) return 85.0;
        return 100.0 - (100.0 / (1.0 + (ag / al)));
      }
      return [calc(gains.slice(0, -1), losses.slice(0, -1)), calc(gains.slice(-(period || 6)), losses.slice(-(period || 6)))];
    }

    function computeBollinger(prices) {
      if (prices.length < 3) {
        var p = prices.length ? prices[prices.length - 1] : 0.5;
        return { low: p, mid: p, up: p, width: 0.0, squeeze: false };
      }
      var mid = prices.reduce(function (a, v) { return a + v; }, 0) / prices.length;
      var variance = prices.reduce(function (a, v) { return a + Math.pow(v - mid, 2); }, 0) / prices.length;
      var sd = Math.sqrt(variance);
      var up = mid + 2.0 * sd;
      var low = mid - 2.0 * sd;
      var width = up - low;
      return { low: low, mid: mid, up: up, width: width, squeeze: width < 0.045 };
    }

    function computeMacdProxy(prices) {
      if (prices.length < 4) return { macd: 0.0, hist: 0.0, crossUp: false, crossDown: false };
      var fast = computeEma(prices, 3);
      var slow = computeEma(prices, 6);
      var macdLine = fast.map(function (f, idx) { return f - slow[idx]; });
      var sigLine = computeEma(macdLine, 3);
      var histCurr = macdLine[macdLine.length - 1] - sigLine[sigLine.length - 1];
      var histPrev = macdLine.length >= 2 ? (macdLine[macdLine.length - 2] - sigLine[sigLine.length - 2]) : histCurr;
      return {
        macd: macdLine[macdLine.length - 1],
        hist: histCurr,
        crossUp: histPrev <= 0.0 && histCurr > 0.0,
        crossDown: histPrev >= 0.0 && histCurr < 0.0
      };
    }

    function computeStochasticProxy(prices, highs, lows) {
      if (prices.length < 3) return 50.0;
      var hh = Math.max.apply(null, highs.slice(-6));
      var ll = Math.min.apply(null, lows.slice(-6));
      var denom = Math.max(hh - ll, 0.001);
      return Math.max(0.0, Math.min(100.0, ((prices[prices.length - 1] - ll) / denom) * 100.0));
    }

    function analyzeAllStrategies(segs, mode) {
      if (!segs || !segs.length) return null;
      var active = segs.filter(function (s) { return s.total >= 2; });
      if (active.length < 4) return null;

      var prices = active.map(function (s) { return 1.0 - s.avg_y; });
      var highs = active.map(function (s) { return 1.0 - s.top_y; });
      var lows = active.map(function (s) { return 1.0 - s.bot_y; });
      var spans = active.map(function (s) { return s.span_y; });

      var resLevel = Math.max.apply(null, highs);
      var supLevel = Math.min.apply(null, lows);
      var totalRange = Math.max(resLevel - supLevel, 0.04);
      var currP = prices[prices.length - 1];
      var pricePos = Math.max(0.0, Math.min(1.0, (currP - supLevel) / totalRange));

      var lastSeg = active[active.length - 1];
      var prevSeg = active[active.length - 2];
      var prev2Seg = active[active.length - 3];

      var hhHl = 0, lhLl = 0;
      for (var i = 1; i < active.length; i++) {
        if (highs[i] > highs[i - 1] + 0.012 && lows[i] > lows[i - 1] + 0.01) hhHl++;
        else if (highs[i] < highs[i - 1] - 0.012 && lows[i] < lows[i - 1] - 0.01) lhLl++;
      }

      var midIdx = Math.max(1, Math.floor(prices.length / 2));
      var firstHalf = prices.slice(0, midIdx).reduce(function (a, v) { return a + v; }, 0) / midIdx;
      var secondHalf = prices.slice(midIdx).reduce(function (a, v) { return a + v; }, 0) / Math.max(1, prices.length - midIdx);
      var slope = secondHalf - firstHalf;

      var marketStructure = 'SIDEWAYS_RANGE';
      if (totalRange < 0.085 && Math.abs(slope) < 0.025) marketStructure = 'SIDEWAYS_RANGE';
      else if (slope >= 0.025 || hhHl > lhLl) marketStructure = 'UPTREND';
      else if (slope <= -0.025 || lhLl > hhHl) marketStructure = 'DOWNTREND';

      var pullbackUp = marketStructure === 'UPTREND' && prices[prices.length - 2] < prices[prices.length - 3] && prices[prices.length - 1] > prices[prices.length - 2];
      var pullbackDown = marketStructure === 'DOWNTREND' && prices[prices.length - 2] > prices[prices.length - 3] && prices[prices.length - 1] < prices[prices.length - 2];
      var contUp = marketStructure === 'UPTREND' && prices[prices.length - 1] > prices[prices.length - 2] && prices[prices.length - 2] > prices[prices.length - 3];
      var contDown = marketStructure === 'DOWNTREND' && prices[prices.length - 1] < prices[prices.length - 2] && prices[prices.length - 2] < prices[prices.length - 3];

      var emaFast = computeEma(prices, 4);
      var slowSlice = prices.slice(-8);
      var smaSlowVal = slowSlice.reduce(function (a, v) { return a + v; }, 0) / Math.max(1, slowSlice.length);
      var prevSlowSlice = prices.slice(-9, -1);
      var prevSmaVal = prevSlowSlice.length ? (prevSlowSlice.reduce(function (a, v) { return a + v; }, 0) / prevSlowSlice.length) : smaSlowVal;

      var maBull = emaFast[emaFast.length - 1] > smaSlowVal && currP >= emaFast[emaFast.length - 1];
      var maBear = emaFast[emaFast.length - 1] < smaSlowVal && currP <= emaFast[emaFast.length - 1];
      var maCrossUp = emaFast.length >= 2 && emaFast[emaFast.length - 2] <= prevSmaVal && emaFast[emaFast.length - 1] > smaSlowVal;
      var maCrossDown = emaFast.length >= 2 && emaFast[emaFast.length - 2] >= prevSmaVal && emaFast[emaFast.length - 1] < smaSlowVal;

      var rsiPair = computeVisualRsi(prices, 6);
      var prevRsi = rsiPair[0], currRsi = rsiPair[1];
      var rsiOversold = currRsi <= 32.0;
      var rsiOverbought = currRsi >= 68.0;
      var bullDiv = currP < Math.min.apply(null, prices.slice(0, -1)) && currRsi > prevRsi + 4.0;
      var bearDiv = currP > Math.max.apply(null, prices.slice(0, -1)) && currRsi < prevRsi - 4.0;

      var bb = computeBollinger(prices.slice(-10));
      var bbTouchLow = currP <= bb.low + (bb.width * 0.12);
      var bbTouchUp = currP >= bb.up - (bb.width * 0.12);
      var bbVolExpand = bb.width >= 0.16;
      var atrSlice = spans.slice(-5);
      var atrEst = atrSlice.reduce(function (a, v) { return a + v; }, 0) / Math.max(1, atrSlice.length);
      var atrExpand = lastSeg.span_y > atrEst * 1.35 && lastSeg.span_y >= 0.08;

      var macd = computeMacdProxy(prices);
      var stochK = computeStochasticProxy(prices, highs, lows);
      var stochOs = stochK <= 25.0;
      var stochOb = stochK >= 75.0;

      var eqHighSweep = highs[highs.length - 1] >= Math.max.apply(null, highs.slice(0, -1)) - 0.005 && pricePos >= 0.75;
      var eqLowSweep = lows[lows.length - 1] <= Math.min.apply(null, lows.slice(0, -1)) + 0.005 && pricePos <= 0.25;

      var fvgUp = lows[lows.length - 1] > highs[highs.length - 3] + 0.01 && prices[prices.length - 1] > prices[prices.length - 2];
      var fvgDown = highs[highs.length - 1] < lows[lows.length - 3] - 0.01 && prices[prices.length - 1] < prices[prices.length - 2];

      var bullOb = pricePos <= 0.35 && prices[prices.length - 1] > prices[prices.length - 2] && prev2Seg.bear > prev2Seg.bull;
      var bearOb = pricePos >= 0.65 && prices[prices.length - 1] < prices[prices.length - 2] && prev2Seg.bull > prev2Seg.bear;

      var insideBar = lastSeg.span_y < prevSeg.span_y * 0.65 && prevSeg.span_y >= 0.06;
      var tweezerTop = pricePos >= 0.60 && Math.abs(highs[highs.length - 1] - highs[highs.length - 2]) <= 0.02;
      var tweezerBot = pricePos <= 0.40 && Math.abs(lows[lows.length - 1] - lows[lows.length - 2]) <= 0.02;

      var ptot = Math.max(prevSeg.bull + prevSeg.bear, 1);
      var ltot = Math.max(lastSeg.bull + lastSeg.bear, 1);
      var t4 = Math.max(prev2Seg.bull + prev2Seg.bear, 1);

      var pBull = prevSeg.bull / ptot;
      var pBear = prevSeg.bear / ptot;
      var lBull = lastSeg.bull / ltot;
      var lBear = lastSeg.bear / ltot;

      var upperWick = Math.max(0.0, lastSeg.avg_y - lastSeg.top_y);
      var lowerWick = Math.max(0.0, lastSeg.bot_y - lastSeg.avg_y);

      var strongUpperWick = upperWick > lowerWick * 1.35 && lastSeg.span_y >= 0.06;
      var strongLowerWick = lowerWick > upperWick * 1.35 && lastSeg.span_y >= 0.06;

      var hammer = pricePos <= 0.38 && (strongLowerWick || (lBull >= 0.56 && lowerWick > upperWick));
      var shootingStar = pricePos >= 0.62 && (strongUpperWick || (lBear >= 0.56 && upperWick > lowerWick));

      var bullMarubozu = lBull >= 0.70 && !strongUpperWick;
      var bearMarubozu = lBear >= 0.70 && !strongLowerWick;

      var bullEngulf = pBear >= 0.52 && lBull >= 0.58 && lastSeg.bull >= prevSeg.bear;
      var bearEngulf = pBull >= 0.52 && lBear >= 0.58 && lastSeg.bear >= prevSeg.bull;

      var s5Small = (prevSeg.total <= Math.max(prev2Seg.total, lastSeg.total) * 0.85) || (Math.abs(pBull - pBear) < 0.25);
      var morningStar = (prev2Seg.bear / t4 >= 0.54) && s5Small && (lBull >= 0.56) && (pricePos <= 0.48);
      var eveningStar = (prev2Seg.bull / t4 >= 0.54) && s5Small && (lBear >= 0.56) && (pricePos >= 0.52);

      var recTotal = active.slice(-3).reduce(function (acc, s) { return acc + s.bull + s.bear; }, 0);

      var upScore = 0.0, downScore = 0.0, upTags = [], downTags = [];

      if (marketStructure === 'UPTREND') { upScore += 0.85; upTags.push('Uptrend'); }
      else if (marketStructure === 'DOWNTREND') { downScore += 0.85; downTags.push('Downtrend'); }

      if (pullbackUp) { upScore += 0.75; upTags.push('PullbackUp'); }
      if (pullbackDown) { downScore += 0.75; downTags.push('PullbackDown'); }
      if (contUp) { upScore += 0.70; upTags.push('ContUp'); }
  
