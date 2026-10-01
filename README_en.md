"auto";
auto.waitFor();
if (!requestScreenCapture()) { toast("Screen capture permission do"); exit(); }
setScreenMetrics(1080, 2400);

var GAME_NAME = "Freezer Frenzy";
var GAME_PKG = getPackageName(GAME_NAME);
if (!GAME_PKG) { toast("Game nahi mila. GAME_NAME check kar"); exit(); }

var W = 1080;
var SCAN_FROM = 480, SCAN_TO = 1700;
var STEP = 24;
var SAFE = 230;
var OBST = [228, 45, 0], OBST_TOL = 38, OBST_MIN = 12;
var ROPE = [192, 129, 16], ROPE_TOL = 30;
var ROPE_Y = 1150;
var DRAG_Y = 2000;
var HOME_TAP = [540, 1350];
var POPUP_WAIT = 12000;
var CLOSE_TAPS = [[540, 1790], [540, 1880]];

events.observeKey();
events.onKeyDown("volume_down", function () {
    toast("Bot stop");
    engines.myEngine().forceStop();
});

function log(m) { console.log("[FreezerBot] " + m); }

function near(c, t, tol) {
    var dr = colors.red(c) - t[0], dg = colors.green(c) - t[1], db = colors.blue(c) - t[2];
    return dr * dr + dg * dg + db * db < tol * tol;
}

function getState(img) {
    var c = images.pixel(img, 100, 1900);
    var r = colors.red(c), g = colors.green(c), b = colors.blue(c);
    if (r < 100 && g < 100 && b < 115) return "popup";
    if (r > 225 && g > 150 && b < 140) return "home";
    if (b > 190 && g > 170 && r < 235) return "game";
    return "other";
}

function getHookX(img, last) {
    var xs = [];
    for (var x = 20; x < 1060; x += 3) {
        if (near(images.pixel(img, x, ROPE_Y), ROPE, ROPE_TOL)) xs.push(x);
    }
    if (!xs.length) return last;
    return xs[Math.floor(xs.length / 2)];
}

function getObstacleXs(img) {
    var xs = [];
    for (var y = SCAN_FROM; y < SCAN_TO; y += STEP) {
        for (var x = 0; x < W; x += STEP) {
            if (near(images.pixel(img, x, y), OBST, OBST_TOL)) xs.push(x);
        }
    }
    return xs.length >= OBST_MIN ? xs : [];
}

function pickTarget(hx, xs) {
    function minDist(x) {
        var m = 1e9;
        for (var i = 0; i < xs.length; i++) m = Math.min(m, Math.abs(xs[i] - x));
        return m;
    }
    if (minDist(hx) >= SAFE) return hx;
    var best = null, bd = 1e9, x;
    for (x = 150; x <= 930; x += 30) {
        if (minDist(x) >= SAFE && Math.abs(x - hx) < bd) { best = x; bd = Math.abs(x - hx); }
    }
    if (best !== null) return best;
    var bx = hx, bm = -1;
    for (x = 150; x <= 930; x += 30) {
        var m = minDist(x);
        if (m > bm) { bm = m; bx = x; }
    }
    return bx;
}

function dragTo(hx, tx) {
    var d = Math.abs(tx - hx);
    if (d < 40) return;
    gesture(Math.min(400, 120 + d * 0.4), [hx, DRAG_Y], [tx, DRAG_Y]);
}

function tryAdClose() {
    var w = textMatches(/^(X|x|\u00d7|\u2715|\u2716|Close|CLOSE|close|Skip|SKIP|Skip Ad|Skip ad)$/).findOnce() ||
            descMatches(/^(Close|close|CLOSE|Skip|X|Dismiss)$/).findOnce();
    if (w) {
        var b = w.bounds();
        click(b.centerX(), b.centerY());
        log("Ad close mila, click kiya");
        return true;
    }
    return false;
}

function relaunch() {
    log("Game relaunch");
    app.launchPackage(GAME_PKG);
    sleep(4000);
}

var lastX = 540, popupSince = 0, otherSince = 0, lastBack = 0, closeIdx = 0;

while (true) {
    try {
        var now = Date.now();
        var pkg = currentPackage();

        if (pkg && pkg !== GAME_PKG && pkg.indexOf("launcher") < 0 && pkg.indexOf("systemui") < 0) {
            if (otherSince === 0) otherSince = now;
            if (!tryAdClose()) {
                if (now - lastBack > 8000) { back(); lastBack = now; log("back()"); }
            }
            if (now - otherSince > 100000) { relaunch(); otherSince = 0; }
            sleep(1500);
            continue;
        }

        var img = captureScreen();
        if (!img) { sleep(300); continue; }
        var st = getState(img);

        if (st === "game") {
            popupSince = 0; otherSince = 0;
            var hx = getHookX(img, lastX);
            lastX = hx;
            var xs = getObstacleXs(img);
            img.recycle();
            if (xs.length) {
                var tx = pickTarget(hx, xs);
                dragTo(hx, tx);
            }
            sleep(40);

        } else if (st === "home") {
            img.recycle();
            popupSince = 0; otherSince = 0;
            log("Home: TAP TO DROP");
            click(HOME_TAP[0], HOME_TAP[1]);
            sleep(1800);

        } else if (st === "popup") {
            img.recycle();
            otherSince = 0;
            if (popupSince === 0) { popupSince = now; log("Popup, wait..."); }
            var el = now - popupSince;
            if (el < POPUP_WAIT) {
                sleep(500);
            } else {
                var p = CLOSE_TAPS[closeIdx % CLOSE_TAPS.length];
                closeIdx++;
                click(p[0], p[1]);
                sleep(1500);
                if (el > 120000) { back(); popupSince = now; }
            }

        } else {
            img.recycle();
            if (otherSince === 0) otherSince = now;
            var oe = now - otherSince;
            if (oe > 3000) {
                if (!tryAdClose() && oe > 15000 && now - lastBack > 10000) {
                    back(); lastBack = now; log("back() (ad)");
                }
            }
            if (oe > 120000) { relaunch(); otherSince = 0; }
            sleep(800);
        }
    } catch (e) {
        log("Error: " + e);
        sleep(2000);
    }
}
