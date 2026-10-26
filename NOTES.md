# NOTES (GTA III / re3 VR port)

## Base (Etap 0)
- Fork: `codepdbh/re3-android-port-evolved`, branch `master`, base commit `0219ec7d` (re3 / GTA III, Android + SDL2 + touch overlay). Same author and layout as the reVC fork used in `../src`.
- Not usable: `halpz/re3` and `GTAmodding/re3` (DMCA takedown, HTTP 403). `mrxenginner/re3` is desktop-only (GLFW, no Android), rejected.
- Licence: see `LICENSE.md` / `README.md` of the fork (checked in `../src`: MIT with Proof-of-Usage). Not re-checked for this repo.
- librw submodule: `vendor/librw` @ `fb5ede8` (reVC fork uses `0ad67ee` + patches `../src/patches/librw-*.patch`; they will have to be rebased on `fb5ede8`).
- Package `com.re3.game` (different from reVC's `com.revc.game`, both can be installed), data folder `/sdcard/re3GTA`.

## Build
- Gradle 8.11.1, NDK `27.2.12479018`, ndk/cmake as in `android/launcher/app/build.gradle`. JDK 17.
- `android/launcher/local.properties`: `sdk.dir=$HOME/Android/Sdk` (not committed).
- `cd android/launcher && JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64 sh gradlew assembleDebug` -> builds with no changes (44 s, BUILD SUCCESSFUL). Flavours `standard` and `fwr` (Frosted Winter Remastered mod, own folder `re3GTA_FWR`).
- APK: `android/launcher/app/build/outputs/apk/standard/debug/app-standard-debug.apk`.
- 16 KB pages: `jniLibs.useLegacyPackaging = true` already set.

## Game data
- Pushed with `adb push "GAMES/Grand Theft Auto 3" /sdcard/re3GTA` (1.2 GB on the phone, 5 min). Never commit game files.

## Test log
| Stage | Where | Result |
|---|---|---|
| 0 build | host | assembleDebug OK |
| 0 install/launch | Redmi Note 15 Pro+ (2510ERA8BG), Android 15, 2772x1280, adb over network | first install blocked by MIUI (`INSTALL_FAILED_USER_RESTRICTED`, needs on-phone confirmation); second attempt accepted, installs, loads data from `/sdcard/re3GTA`, main menu shown with touch overlay, no crash in logcat (20 s) |
| 0 cutout | phone (screenshot) | menu spans the full width. `shortEdges` is set in `GameActivity.java` (~line 89). Not confirmed via a window-size log. |

Not tested: gameplay, FPS, 16 KB page device, glasses.

## Etap 1: gamepad (GameSir T4n Lite, "GameSir-Nova Lite", BT)
- Bug on the phone (user): pressing a pad button crashed the game (`re_assert "false" failed`, from `assert(gamepad1 != nullptr)` in `joysChangeCB` when SDL has no mapping). Same bug as in reVC.
- Ported the reVC stage 1 changes to `skel/sdl2/sdl2.cpp` and `core/config.h`: `CapturePad` polls `SDL_NumJoysticks` about once a second, `AndroidEnsureControllerMapping` registers the default Android layout, `joysChangeCB` no longer aborts when opening fails and skips `uinput-xiaomi`, `CIRCLE_BACK_BUTTON` on Android. Log tag `re3 pad:`.
- Log after the change (phone): device 1 'GameSir-Nova Lite' detected, `isGameController 1`, no abort at launch. Phone (user): pad works (menu and game).
- Not ported (VC-only): map marker/Cross handling and the Bike drive-by note.

## Etap 2: "cinema" presentation (VR_SBS, scene rendered once)
- Ported from reVC (stage 2 state): `src/skel/vr/vr.{h,cpp}` (log tag `re3vr`), hooks in `rw/RwHelper.cpp` (`CameraSize`) and `skel/sdl2/sdl2.cpp` (`psCameraShowRaster`), `-DVR_SBS` in `app/build.gradle`. Same design: the camera gets a one-eye FBO raster, so `RsGlobal.width/height` = one eye; `VR_Present()` draws it twice with a 32x32 lens mesh, shift and margin.
- Config: `/sdcard/re3GTA/vr.ini`, polled twice a second, live (`adb push vr.ini /sdcard/re3GTA/`). Keys: `VR_Enabled`, `VR_LensEnabled`, `VR_LensK1`, `VR_LensK2`, `VR_Shift`, `VR_Margin`, `VR_EyeRenderScale`, `VR_EyeAspect`, `VR_Uncapped`. Defaults in code are the reVC stage 2 starting values (k1 0.176, k2 0.192), not tuned for GTA III.
- librw: `patches/librw-vr.patch` (camera texture in `rasterRenderFast`) and `patches/librw-gl3-immediate-buffers.patch` (rebased on `fb5ede8`, applied with `patch -p1`, fuzz 2) must be applied to `vendor/librw` before building (`git apply` / `patch -p1`). The submodule working tree carries them uncommitted.
- Not ported yet: menu page "VR Settings" (stage 5 of the plan), stereo, head tracking.

| Check | Where | Result |
|---|---|---|
| VR off (`VR_Enabled = 0`, live) | phone (screenshot) | flat menu as before |
| VR on, shift 0, lens off, margin 0: mean abs diff L/R (menu region without touch buttons, screencap) | phone | 0.00005 (whole screen 4.8, caused only by the touch overlay drawn differently in each half) |
| shift 0.15, margin 0.1, lens on k1 0.176 k2 0.192 | phone (screenshot only) | images moved toward centre, black margin, barrel shape, live without restart; tuning not verified |
| FPS (menu, limiter on) | phone | 60 (menu cap) |

| In-game scene, VR on, shift 0, lens off, scale 0.75 (VR switched on live during play) | phone (screenshot) | world, HUD, radar and text in both halves; mean abs diff L/R 0.00001 |
| FPS in game, city, on foot beside a car, scale 0.75 | phone | 103-145 (the fork's frame limiter was not 30 in this run; 5 s samples, noisy) |

Not verified: gameplay FPS under load, comfort, k1/k2, shift, margin (glasses). `vr.ini` on the phone is left with `VR_Enabled = 0`.

## Etap 3: real stereo (`VR_Mode = stereo`)
- Ported from reVC (stage 3 plus the later stereo fixes). `VR_Mode = stereo`, `VR_Separation` (m), `VR_Convergence` (m), `VR_FovScale` (default 1.0 = game FOV; reVC goggles used 1.5, not tuned for GTA III). `cinema` keeps stage 2.
- `core/main.cpp`: the render part of `Idle()` is now `RenderFramePass(firstEye)`. In stereo (`VR_Active() && VR_StereoFrame()`, no menu, no full fade) it runs twice: `VR_BeginEye(0/1)` switches the camera raster to the left/right eye texture and applies the off-axis camera (frame shifted by T = E(1-1/C), `viewOffset.x = E/C`, E = +-separation/2). The second pass skips `ConstructRenderList`/`PreRender`/`ScreenDroplets::Process`. re3's condition `m_bRenderGameInMenu` is not stereo (menus stay single pass, same image in both eyes). HUD, radar and text are drawn in each pass with the same layout (zero parallax).
- Per-frame state that must not run twice: stored shadows (`CShadows::RenderStoredShadows`, `VR_HoldFrameLists`), corona streak history (`CCoronas::Render`), random numbers (`VR_GetRandState/VR_SetRandState` in `core/re3.cpp`, restored before the second eye), `CPostFX` motion blur needing the previous frame is off during the stereo passes.
- Not needed in re3 (checked): `CSprite::CalcScreenCoors` shear (reVC had a double shear and it was removed again; `cam->viewMatrix` already contains `viewOffset`).
- Not ported / not checked in re3: `CMBlur` (`CCamera::RenderMotionBlur`, `MBlur.cpp`) keeps one front buffer for both eyes; `CPostFX::SmoothColor` does not exist in GTA III. Reflections, near weapon/vehicle, droplets not examined.
- Test hint: `adb push` can be read by the game half-written and the 1-second `mtime` then hides the final write; run `adb shell touch /sdcard/re3GTA/vr.ini` after pushing (the first 0.25 test missed this and showed 0.0002).

| Check | Where | Result |
|---|---|---|
| menu, stereo, separation 0, L/R mean abs diff | phone (screencap) | 0.003 |
| in game (city street, on foot), stereo, separation 0, shift 0, lens off | phone | 0.0004 |
| same, separation 0.25, convergence 6 | phone | 6.0; screenshot: lamp post and cars shifted between halves, HUD and radar identical |
| FPS in game, stereo 0.25, scale 0.75, `VR_Uncapped = 1`, standing on a quiet street | phone | 90-111 (cinema in the same place was 103-145) |

- Phone with glasses (comfort, depth, flicker of coronas/shadows/rain in one eye, motion blur, near weapon/vehicle): not checked yet.
- Head tracking, VR settings menu: not ported yet (stages 4 and 5).

## Etap 4: head tracking (`VR_HeadTracking = 1`)
- Ported from reVC: `HeadTracker.java` (package `com.re3.game`, started/stopped in `GameActivity.onResume/onPause`; `TYPE_GAME_ROTATION_VECTOR`, remapped for display rotation) -> JNI `Java_com_re3_game_HeadTracker_nativeHeadQuat` in `skel/vr/vr.cpp`. Head basis (yaw relative to the recenter reference, pitch and roll from gravity) recomputed once per frame in `VR_Poll`.
- Applied only to rendering: `VR_BeginEye(eye)` builds the render camera frame from the game camera matrix (`TheCamera.GetMatrix()`, the inverse of `m_cameraMatrix`) combined with the head basis; eye -1 = cinema with head tracking, 0/1 = stereo. Game camera, aiming and walking are untouched.
- Culling follows the head: `CCamera::IsSphereVisible` and `IsBoxVisible` (`core/Camera.cpp`) call `VR_HeadToView()` before the frustum tests.
- Recenter: L3 + R3 on the pad (edge triggered, `CapturePad`), first sensor sample also recenters. The menu item comes with the VR menu (stage 5, not ported yet). `VR_RollScale` (default 0.5), `VR_FovScale` (default 1.0) in `vr.ini`. Default `VR_HeadTracking` is off.
- Log: `head: tracking .. active .. sensor .. yaw/pitch/roll` every 5 s (tag `re3vr`).
- Phone (no glasses): app starts with the sensor, no `UnsatisfiedLinkError`; log shows sensor valid and yaw/pitch/roll changing when the phone is moved (e.g. `yaw -64 pitch -13 roll 2`). Direction of the view turn, culling at the edges of the turned view, recenter and drift: not checked, need the phone in the goggles. The yaw sign was flipped in reVC after a goggle test; the same sign is used here, unverified for GTA III.

## Etap 5: VR options menu on the pad, keep screen on, sustained performance
- Menu: Options -> "VR Settings" (`MENUPAGE_VR_SETTINGS`, `core/Frontend.h`, screen in `core/MenuScreensCustom.cpp`, 14 `CFO_DYNAMIC` rows, `VR_MenuDraw/VR_MenuPress` in `skel/vr/vr.cpp`): VR on/off, mode (cinema/stereo), separation, convergence, lens on/off, K1, K2, shift, margin, render scale, field of view scale, head tracking, head roll scale, recenter head. D-pad left/right changes the value, every change is written to `vr.ini` and applies live. Texts via `VR_Text()` (keys `VRM_*`) hooked at the top of `CText::Get` (`text/Text.cpp`), no GXT edit. Page uses the small font (`SMALLESTTEXT_*`) so 15 rows fit; the list scrolls.
- `GameActivity.onCreate`: `FLAG_KEEP_SCREEN_ON`, `setSustainedPerformanceMode(true)` when `PowerManager.isSustainedPerformanceModeSupported()`. Whether the Redmi reports support was not checked, and the screen staying on over a long idle was not observed.
- Phone (user, no glasses): the page opens, shows the rows (screenshot, both eyes), values change with the pad (log: lens switched on, head tracking switched off from the menu). Not yet looked at: the last rows (roll scale, recenter), Back from the page, saving after restart.
- Not measured for GTA III: long thermal/FPS session (20+ min) in stereo; only short FPS samples (90-185 FPS, standing, uncapped). Comfort, k1/k2, shift, margin, FOV scale and head tracking directions need real glasses.

### Poprawka: światła uliczne/dla pieszych tylko w lewym oku
`CBrightLights::Render` zerował `NumBrightLights` po pierwszym oku, więc drugie nic nie rysowało (żarówki sygnalizacji i świateł dla pieszych). Teraz lista jest trzymana przez `VR_HoldFrameLists()`. Zgłoszone przez użytkownika, naprawa zbudowana i zainstalowana, jeszcze niezweryfikowana na ekranie.

To samo dla `CShinyTexts` (DONT WALK / WALK): `NumShinyTexts` było zerowane po pierwszym oku; teraz `VR_HoldFrameLists()`. Zbudowane i zainstalowane, niezweryfikowane na ekranie.

To samo dla `CPickups::RenderPickUpText` (ceny broni w Ammu-Nation): `NumMessages` zerowane po pierwszym oku, teraz `VR_HoldFrameLists()`. Pozycje tekstu liczone są w PreRender pierwszego oka. Niezweryfikowane na ekranie.

To samo dla `CGlass::Render`: `NumGlassEntities` (szyby w oknach budynków) zerowane po pierwszym oku, teraz `VR_HoldFrameLists()`. Przejrzane też Antennas, Rubbish, Skidmarks, WeaponEffects, Particle: nie zerują list w Render. Niezweryfikowane na ekranie.

## Znany problem: jasne szczeliny na granicach terenu (nierozwiązany)
Na granicach trawnik/chodnik/jezdnia widać cienką, kropkowaną jasną linię ("mapa cieknie"). Zgłoszone przez użytkownika w goglach.
- Przyczyna: T-junctions w geometrii mapy oryginalnej gry; przez szczeliny widać tło, czyli gradient nieba z `DoRWStuffStartOfFrame` (jasny przy horyzoncie).
- Zrzuty z telefonu (bez okularów): ta sama linia jest w trybie płaskim (VR wyłączony), w kinie i w stereo; `VR_EyeRenderScale` 0.5/0.75/1.0 nie zmienia jej. Soczewki powiększają efekt. Nie porównano ilościowo stereo vs płaski (różne klatki).
- Decyzja: nie naprawiać. Ewentualna opcja na później: ciemniejszy kolor czyszczenia bufora w trybie VR (obejście, zmienia wygląd horyzontu).

## Publication copy
This repository is a snapshot of the development tree published as a single initial commit (no history). Left out on purpose, because they come from the original game or its artwork: `gamefiles/` (textures `.txd`, texts `.gxt`, scripts `.scm`, data tables), upstream screenshots and icon, `src/skel/win/gta3.ico`, `gtavc.ico`, `icono/`. The upstream README is kept as `docs/UPSTREAM_README.md` (Spanish, for attribution; its images are gone). The launcher icon is original (`tools/make_icons.py`). The earlier commit log lives in the private development repositories, so commit hashes quoted in the notes above refer to those.
