            #include "raylib.h"
            #include <stdlib.h>
            #include <stdbool.h>
            #include <time.h>
            #include <math.h>
            #include <stdio.h>
            
            #define SCREEN_WIDTH 800
            #define SCREEN_HEIGHT 350
            #define GROUND_Y 270
            #define MAX_BG_PARTICLES 80
            #define MAX_FX_PARTICLES 200
            #define MAX_FLOAT_TEXTS 15
            
            typedef enum {
                STATE_TITLE = 0,
                STATE_PLAYING,
                STATE_CUTSCENE,
                STATE_GAMEOVER
            } GameState;
            
            // 무기 종류 (점진적 해금 및 성장 시스템)
            typedef enum {
                WEAPON_DAGGER = 0,    // 기본 단검 (초기 상태)
                WEAPON_GREATSWORD,    // 대검 (중급 해금)
                WEAPON_DUALBLADES     // 쌍검 (고급 해금)
            } WeaponType;
            
            typedef struct {
                float x, y;
                float speed;
                float size;
                Color color;
            } BgParticle;
            
            typedef struct {
                float x, y;
                float vx, vy;
                float alpha;
                float size;
                Color color;
                bool active;
            } FxParticle;
            
            typedef struct {
                char text[48];
                float x, y;
                float alpha;
                Color color;
                bool active;
            } FloatText;
            
            typedef struct {
                float x, y;
                float width, height;
                float vy;
                bool isJumping;
                bool isSliding;
                bool isSlashing;
                float slashAnimTimer;
                float slashAngle;
                float squashFactorX;
                float squashFactorY;
                int animTimer;
                int legFrame;
                WeaponType currentWeapon;
            } Dino;
            
            typedef struct {
                float x, y;
                float width, height;
                bool active;
                int type;
            } Obstacle;
            
            typedef struct {
                float x;
                float width;
                bool active;
                float timer;
            } UltimateBeam;
            
            const Color BALANCED_PALETTE[6] = {
                { 0, 180, 255, 255 },   // 네온 블루
                { 50, 255, 100, 255 },  // 네온 그린
                { 255, 230, 0, 255 },   // 네온 옐로우
                { 255, 100, 0, 255 },   // 네온 오렌지
                { 255, 50, 150, 255 },  // 네온 핑크
                { 255, 255, 255, 255 }  // 화이트
            };
            
            float shakeTimer = 0.0f;
            float shakeIntensity = 0.0f;
            float flashTimer = 0.0f;
            float cutsceneTimer = 0.0f;
            float cutsceneMaxTime = 6.0f; // 깔끔하고 묵직한 6초 컷씬
            
            BgParticle bgParticles[MAX_BG_PARTICLES];
            FxParticle fxParticles[MAX_FX_PARTICLES];
            FloatText floatTexts[MAX_FLOAT_TEXTS];
            UltimateBeam ultBeam = { 0, 0, false, 0 };
            
            bool isGodMode = false;
            
            void TriggerShake(float duration, float intensity) {
                shakeTimer = duration;
                shakeIntensity = intensity;
            }
            
            void TriggerFlash(float duration) {
                flashTimer = duration;
            }
            
            void SpawnFloatText(const char* text, float x, float y, Color col) {
                for (int i = 0; i < MAX_FLOAT_TEXTS; i++) {
                    if (!floatTexts[i].active) {
                        snprintf(floatTexts[i].text, sizeof(floatTexts[i].text), "%s", text);
                        floatTexts[i].x = x;
                        floatTexts[i].y = y;
                        floatTexts[i].alpha = 1.0f;
                        floatTexts[i].color = col;
                        floatTexts[i].active = true;
                        break;
                    }
                }
            }
            
            void SpawnFx(float x, float y, Color col) {
                for (int k = 0; k < 15; k++) {
                    for (int i = 0; i < MAX_FX_PARTICLES; i++) {
                        if (!fxParticles[i].active) {
                            fxParticles[i].x = x;
                            fxParticles[i].y = y;
                            float angle = (float)(rand() % 360) * 3.14159f / 180.0f;
                            float speed = (float)(80 + rand() % 180);
                            fxParticles[i].vx = cosf(angle) * speed;
                            fxParticles[i].vy = sinf(angle) * speed;
                            fxParticles[i].alpha = 1.0f;
                            fxParticles[i].size = (float)(2 + rand() % 5);
                            fxParticles[i].color = col;
                            fxParticles[i].active = true;
                            break;
                        }
                    }
                }
            }
            
            void InitSystems() {
                for (int i = 0; i < MAX_BG_PARTICLES; i++) {
                    bgParticles[i].x = (float)(rand() % SCREEN_WIDTH);
                    bgParticles[i].y = (float)(rand() % SCREEN_HEIGHT);
                    bgParticles[i].speed = (float)(20 + rand() % 50);
                    bgParticles[i].size = (float)(1 + rand() % 3);
                    bgParticles[i].color = BALANCED_PALETTE[rand() % 6];
                }
                for (int i = 0; i < MAX_FX_PARTICLES; i++) fxParticles[i].active = false;
                for (int i = 0; i < MAX_FLOAT_TEXTS; i++) floatTexts[i].active = false;
            }
            
            void UpdateSystems(float dt) {
                for (int i = 0; i < MAX_BG_PARTICLES; i++) {
                    bgParticles[i].x -= bgParticles[i].speed * dt;
                    if (bgParticles[i].x < 0) {
                        bgParticles[i].x = (float)SCREEN_WIDTH;
                        bgParticles[i].y = (float)(rand() % SCREEN_HEIGHT);
                    }
                }
            
                for (int i = 0; i < MAX_FX_PARTICLES; i++) {
                    if (fxParticles[i].active) {
                        fxParticles[i].x += fxParticles[i].vx * dt;
                        fxParticles[i].y += fxParticles[i].vy * dt;
                        fxParticles[i].alpha -= dt * 2.5f;
                        if (fxParticles[i].alpha <= 0) fxParticles[i].active = false;
                    }
                }
            
                for (int i = 0; i < MAX_FLOAT_TEXTS; i++) {
                    if (floatTexts[i].active) {
                        floatTexts[i].y -= 25.0f * dt;
                        floatTexts[i].alpha -= dt * 1.8f;
                        if (floatTexts[i].alpha <= 0) floatTexts[i].active = false;
                    }
                }
            
                if (shakeTimer > 0) shakeTimer -= dt;
                if (flashTimer > 0) flashTimer -= dt;
            
                if (ultBeam.active) {
                    ultBeam.timer -= dt;
                    ultBeam.width += 2000.0f * dt;
                    if (ultBeam.timer <= 0) {
                        ultBeam.active = false;
                    }
                }
            }
            
            int main(void)
            {
                InitWindow(SCREEN_WIDTH, SCREEN_HEIGHT, "Balanced Dino Run - Homing & Growth");
                SetTargetFPS(60);
            
                srand((unsigned int)time(NULL));
                InitSystems();
            
                GameState state = STATE_TITLE;
                Dino dino = { 80, GROUND_Y - 44, 36, 44, 0, false, false, false, 0.0f, 0.0f, 1.0f, 1.0f, 0, 0, WEAPON_DAGGER };
                Obstacle obstacle = { 0, 0, 0, 0, false, 0 };
            
                float gameSpeed = 7.0f; // 처음엔 편안하고 적절한 속도
                int score = 0;
                int highScore = 0;
                float energy = 0.0f;
                float scoreTimer = 0.0f;
            
                while (!WindowShouldClose())
                {
                    float dt = GetFrameTime();
                    UpdateSystems(dt);
            
                    dino.squashFactorX += (1.0f - dino.squashFactorX) * 10.0f * dt;
                    dino.squashFactorY += (1.0f - dino.squashFactorY) * 10.0f * dt;
            
                    if (dino.slashAnimTimer > 0) {
                        dino.slashAnimTimer -= dt;
                        dino.slashAngle += 20.0f;
                        if (dino.slashAnimTimer <= 0) dino.isSlashing = false;
                    }
            
                    // 점수에 따른 점진적 무기 해금 (성장 요소)
                    if (score >= 400 && score < 1000) dino.currentWeapon = WEAPON_GREATSWORD;
                    else if (score >= 1000) dino.currentWeapon = WEAPON_DUALBLADES;
            
                    switch (state) {
                    case STATE_TITLE:
                        if (IsKeyPressed(KEY_SPACE) || IsKeyPressed(KEY_ENTER)) {
                            state = STATE_PLAYING;
                            dino.y = GROUND_Y - dino.height;
                            dino.vy = 0;
                            dino.isJumping = false;
                            dino.isSliding = false;
                            dino.isSlashing = false;
                            dino.currentWeapon = WEAPON_DAGGER;
                            obstacle.active = false;
                            score = 0;
                            energy = 0.0f;
                            isGodMode = false;
                            gameSpeed = 7.0f;
                        }
                        break;
            
                    case STATE_PLAYING:
                        scoreTimer += dt;
                        if (scoreTimer >= 0.1f) {
                            score += isGodMode ? 5 : 1;
                            scoreTimer = 0.0f;
                            if (!isGodMode && energy < 100.0f) energy += 0.8f;
                        }
            
                        // 점프
                        if ((IsKeyPressed(KEY_SPACE) || IsKeyPressed(KEY_UP) || IsKeyPressed(KEY_W)) && !dino.isJumping && !dino.isSliding) {
                            dino.vy = -13.5f;
                            dino.isJumping = true;
                            dino.squashFactorX = 0.7f;
                            dino.squashFactorY = 1.3f;
                            SpawnFx(dino.x + 10, GROUND_Y, (Color) { 0, 180, 255, 255 });
                        }
            
                        // 슬라이드
                        if ((IsKeyDown(KEY_DOWN) || IsKeyDown(KEY_S)) && !dino.isJumping) {
                            dino.isSliding = true;
                            dino.height = 24;
                            dino.y = GROUND_Y - dino.height;
                        }
                        else if (!dino.isJumping) {
                            dino.isSliding = false;
                            dino.height = 44;
                        }
            
                        // 공격 입력 (J 또는 X 키)
                        if ((IsKeyPressed(KEY_J) || IsKeyPressed(KEY_X)) && !dino.isSlashing) {
                            dino.isSlashing = true;
                            dino.slashAnimTimer = 0.2f;
                            dino.slashAngle = -45.0f;
            
                            // ★ [공중 유도 기능 (Homing Attack)]
                            // 공중인 상태에서 공격을 누르면 가장 가까운 활성 상태의 장애물 위치로 공룡이 부드럽게 유도됨!
                            if (dino.isJumping && obstacle.active) {
                                float targetX = obstacle.x - 30.0f;
                                if (targetX > dino.x && targetX < SCREEN_WIDTH) {
                                    dino.x += (targetX - dino.x) * 0.5f; // 적을 향해 부드럽게 도약 유도
                                    dino.vy = -2.0f; // 체공 시간 유지
                                }
                            }
            
                            // 무기별 사정거리 설정
                            float attackRange = 55.0f;
                            if (dino.currentWeapon == WEAPON_GREATSWORD) attackRange = 85.0f;
                            else if (dino.currentWeapon == WEAPON_DUALBLADES) attackRange = 110.0f;
            
                            Rectangle slashBox = { dino.x + 10, dino.y - 15, attackRange, 70 };
            
                            if (obstacle.active) {
                                Rectangle obstBox = { obstacle.x, obstacle.y, obstacle.width, obstacle.height };
                                if (CheckCollisionRecs(slashBox, obstBox)) {
                                    obstacle.active = false;
                                    score += 200;
                                    energy = fminf(100.0f, energy + 15.0f);
                                    TriggerShake(0.15f, 4.0f);
                                    SpawnFx(obstBox.x + obstBox.width / 2, obstBox.y + obstBox.height / 2, (Color) { 255, 230, 0, 255 });
                                    SpawnFloatText("SLASH! +200", obstBox.x, obstBox.y - 20, (Color) { 255, 230, 0, 255 });
                                }
                            }
                        }
            
                        // 궁극기 각성 진입 ([K] 또는 [C])
                        if ((IsKeyPressed(KEY_K) || IsKeyPressed(KEY_C)) && (energy >= 100.0f || isGodMode)) {
                            energy = 100.0f;
                            state = STATE_CUTSCENE;
                            cutsceneTimer = cutsceneMaxTime;
                            TriggerShake(6.0f, 2.5f);
                        }
            
                        dino.vy += 0.65f;
                        dino.y += dino.vy;
            
                        if (dino.y >= GROUND_Y - dino.height) {
                            dino.y = GROUND_Y - dino.height;
                            if (dino.isJumping) {
                                dino.squashFactorX = 1.3f;
                                dino.squashFactorY = 0.7f;
                            }
                            dino.vy = 0;
                            dino.isJumping = false;
            
                            dino.animTimer++;
                            if (dino.animTimer >= 6) {
                                dino.legFrame = !dino.legFrame;
                                dino.animTimer = 0;
                            }
                        }
            
                        // 장애물 스폰 (충분한 안전 거리 확보)
                        if (!obstacle.active && !ultBeam.active) {
                            obstacle.active = true;
                            obstacle.type = rand() % 2;
                            if (obstacle.type == 0) {
                                obstacle.width = 20;
                                obstacle.height = 38;
                                obstacle.y = GROUND_Y - obstacle.height;
                            }
                            else {
                                obstacle.width = 28;
                                obstacle.height = 22;
                                obstacle.y = GROUND_Y - 70;
                            }
                            obstacle.x = SCREEN_WIDTH + 100; // 스폰 위치 여유분 확보
                        }
            
                        if (obstacle.active) {
                            if (isGodMode) {
                                obstacle.active = false;
                                score += 300;
                                SpawnFx(obstacle.x, GROUND_Y - 30, (Color) { 255, 230, 0, 255 });
                            }
                            else {
                                obstacle.x -= gameSpeed;
                                if (obstacle.x + obstacle.width < 0) obstacle.active = false;
                            }
                        }
            
                        if (!isGodMode) {
                            Rectangle dinoRect = { dino.x + 4, dino.y + 4, dino.width - 8, dino.height - 6 };
                            Rectangle obstRect = { obstacle.x + 4, obstacle.y + 4, obstacle.width - 8, obstacle.height - 8 };
            
                            if (obstacle.active && CheckCollisionRecs(dinoRect, obstRect)) {
                                state = STATE_GAMEOVER;
                                TriggerShake(0.4f, 10.0f);
                                TriggerFlash(0.2f);
                                SpawnFx(dino.x + 15, dino.y + 20, (Color) { 255, 50, 80, 255 });
                                if (score > highScore) highScore = score;
                            }
                        }
                        break;
            
                    case STATE_CUTSCENE:
                        cutsceneTimer -= dt;
                        if (cutsceneTimer <= 0.0f) {
                            state = STATE_PLAYING;
                            isGodMode = true;
                            ultBeam.active = true;
                            ultBeam.x = dino.x + 30;
                            ultBeam.width = 120.0f;
                            ultBeam.timer = 1.0f;
            
                            TriggerShake(1.5f, 25.0f);
                            TriggerFlash(0.8f);
            
                            if (obstacle.active) {
                                obstacle.active = false;
                                score += 5000;
                            }
                        }
                        break;
            
                    case STATE_GAMEOVER:
                        if (IsKeyPressed(KEY_R) || IsKeyPressed(KEY_SPACE)) {
                            state = STATE_PLAYING;
                            dino.x = 80;
                            dino.y = GROUND_Y - dino.height;
                            dino.vy = 0;
                            dino.isJumping = false;
                            dino.isSliding = false;
                            dino.isSlashing = false;
                            dino.currentWeapon = WEAPON_DAGGER;
                            obstacle.active = false;
                            score = 0;
                            energy = 0.0f;
                            isGodMode = false;
                            gameSpeed = 7.0f;
                        }
                        break;
                    }
            
                    float ox = 0, oy = 0;
                    if (shakeTimer > 0) {
                        ox = (float)((rand() % (int)(shakeIntensity * 2)) - shakeIntensity);
                        oy = (float)((rand() % (int)(shakeIntensity * 2)) - shakeIntensity);
                    }
            
                    // --- 렌더링 ---
                    BeginDrawing();
                    ClearBackground((Color) { 10, 10, 18, 255 });
            
                    for (int i = 0; i < MAX_BG_PARTICLES; i++) {
                        DrawCircleV((Vector2) { bgParticles[i].x + ox, bgParticles[i].y + oy }, bgParticles[i].size, Fade(bgParticles[i].color, 0.4f));
                    }
            
                    DrawLine((int)ox, (int)(GROUND_Y + oy), (int)(SCREEN_WIDTH + ox), (int)(GROUND_Y + oy), isGodMode ? (Color) { 255, 230, 0, 255 } : (Color) { 0, 180, 255, 255 });
            
                    if (ultBeam.active) {
                        DrawRectangle((int)(ultBeam.x + ox), (int)(GROUND_Y - 250 + oy), (int)ultBeam.width, 250, Fade((Color) { 255, 50, 200, 255 }, 0.7f));
                        DrawRectangle((int)(ultBeam.x + ox), (int)(GROUND_Y - 150 + oy), (int)ultBeam.width, 100, WHITE);
                    }
            
                    float renderDinoX = dino.x;
                    float renderDinoY = dino.y;
                    float cameraZoom = 1.0f;
            
                    if (state == STATE_CUTSCENE) {
                        float elapsed = cutsceneMaxTime - cutsceneTimer;
                        renderDinoX = SCREEN_WIDTH / 2.0f + sinf(elapsed * 3.0f) * 100.0f;
                        renderDinoY = GROUND_Y - 60.0f + cosf(elapsed * 4.0f) * 15.0f;
                        cameraZoom = 1.2f;
                    }
            
                    Color dinoCol = isGodMode ? (Color) { 255, 255, 255, 255 } : (Color) { 0, 228, 48, 255 };
                    float dx = renderDinoX + ox;
                    float dy = renderDinoY + oy;
            
                    float drawW = dino.width * dino.squashFactorX * cameraZoom;
                    float drawH = dino.height * dino.squashFactorY * cameraZoom;
                    float drawY = dy + (dino.height - drawH);
            
                    DrawRectangleRounded((Rectangle) { dx + 6, drawY + 10, drawW - 12, drawH - 14 }, 0.3f, 4, dinoCol);
                    DrawRectangleRounded((Rectangle) { dx + drawW - 18, drawY, 16 * cameraZoom, 15 * cameraZoom }, 0.3f, 4, dinoCol);
                    DrawCircle((int)(dx + drawW - 10), (int)(drawY + 4), 2, BLACK);
            
                    // 참격 검기 렌더링
                    if (dino.isSlashing) {
                        Color bladeCol = (dino.currentWeapon == WEAPON_DUALBLADES) ? (Color) { 255, 50, 200, 255 } :
                            (dino.currentWeapon == WEAPON_GREATSWORD) ? (Color) { 255, 140, 0, 255 } : (Color) { 0, 255, 255, 255 };
                        DrawRing((Vector2) { dx + 25, drawY + 15 }, 10.0f, 45.0f, dino.slashAngle, dino.slashAngle + 90.0f, 10, Fade(bladeCol, 0.9f));
                    }
                    else if (!dino.isSliding && state != STATE_CUTSCENE) {
                        DrawRectangle((int)(dx + 22), (int)(drawY + 16), 6, 4, dinoCol);
                    }
            
                    if (obstacle.active && state != STATE_CUTSCENE && !isGodMode) {
                        float ox_pos = obstacle.x + ox;
                        float oy_pos = obstacle.y + oy;
            
                        if (obstacle.type == 0) {
                            DrawRectangleRounded((Rectangle) { ox_pos + 4, oy_pos, 14, obstacle.height }, 0.3f, 4, (Color) { 255, 50, 80, 255 });
                        }
                        else {
                            DrawRectangleRounded((Rectangle) { ox_pos, oy_pos + 6, obstacle.width, 12 }, 0.4f, 4, (Color) { 255, 140, 0, 255 });
                        }
                    }
            
                    for (int i = 0; i < MAX_FX_PARTICLES; i++) {
                        if (fxParticles[i].active) {
                            DrawCircleV((Vector2) { fxParticles[i].x + ox, fxParticles[i].y + oy }, fxParticles[i].size, Fade(fxParticles[i].color, fxParticles[i].alpha));
                        }
                    }
                    for (int i = 0; i < MAX_FLOAT_TEXTS; i++) {
                        if (floatTexts[i].active) {
                            DrawText(floatTexts[i].text, (int)(floatTexts[i].x + ox), (int)(floatTexts[i].y + oy), 16, Fade(floatTexts[i].color, floatTexts[i].alpha));
                        }
                    }
            
                    DrawText(TextFormat("HI %05d   %05d", highScore, score), SCREEN_WIDTH - 250, 25, 20, RAYWHITE);
            
                    // 현재 무기 상태 표시
                    const char* weaponNames[3] = { "WEAPON: [DAGGER]", "WEAPON: [GREATSWORD]", "WEAPON: [DUAL BLADES]" };
                    DrawText(weaponNames[dino.currentWeapon], SCREEN_WIDTH - 250, 50, 11, (Color) { 255, 230, 0, 255 });
            
                    DrawText("ENERGY", 25, 20, 12, (Color) { 150, 150, 180, 255 });
                    DrawRectangleRounded((Rectangle) { 25, 38, 150, 14 }, 0.3f, 4, (Color) { 20, 20, 35, 255 });
                    DrawRectangleRounded((Rectangle) { 25, 38, 150 * (energy / 100.0f), 14 }, 0.3f, 4, isGodMode ? (Color) { 255, 230, 0, 255 } : (Color) { 0, 180, 255, 255 });
                    if (energy >= 100.0f && !isGodMode) {
                        DrawText("READY! [K / C]", 185, 38, 12, (Color) { 255, 230, 0, 255 });
                    }
            
                    DrawText("CONTROLS: [SPACE] Jump | [S] Slide | [J/X] Homing Slash | [K/C] Ult", 25, SCREEN_HEIGHT - 25, 11, (Color) { 140, 140, 170, 255 });
            
                    if (state == STATE_CUTSCENE) {
                        float elapsed = cutsceneMaxTime - cutsceneTimer;
                        DrawRectangle(0, 0, SCREEN_WIDTH, 80, BLACK);
                        DrawRectangle(0, SCREEN_HEIGHT - 80, SCREEN_WIDTH, 80, BLACK);
                        DrawText("ASCENDING TO GOD FORM...", SCREEN_WIDTH / 2 - 130, 30, 16, (Color) { 255, 230, 0, 255 });
                    }
            
                    if (flashTimer > 0) {
                        DrawRectangle(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT, Fade(WHITE, flashTimer * 3.0f));
                    }
            
                    if (state == STATE_TITLE) {
                        DrawRectangleRounded((Rectangle) { 150, 95, 500, 160 }, 0.2f, 4, (Color) { 14, 14, 24, 245 });
                        DrawRectangleRoundedLines((Rectangle) { 150, 95, 500, 160 }, 0.2f, 4, (Color) { 0, 180, 255, 255 });
                        DrawText("BALANCED DINO RUN - HOMING EDITION", 165, 115, 17, (Color) { 255, 230, 0, 255 });
                        DrawText("[J/X]: Air Homing Slash | [S]: Slide | [K/C]: Ultimate", 155, 150, 12, (Color) { 50, 255, 100, 255 });
                        DrawText("PRESS [SPACE] TO START", 295, 200, 14, RAYWHITE);
                    }
                    else if (state == STATE_GAMEOVER) {
                        DrawRectangle(0, 0, SCREEN_WIDTH, SCREEN_HEIGHT, Fade(BLACK, 0.75f));
                        DrawRectangleRounded((Rectangle) { 260, 105, 280, 140 }, 0.2f, 4, (Color) { 20, 20, 32, 255 });
                        DrawText("MISSION FAILED", 305, 135, 20, (Color) { 255, 50, 80, 255 });
                        DrawText("Press [R] or [SPACE] to Retry", 280, 185, 14, RAYWHITE);
                    }
            
                    EndDrawing();
                }
            
                CloseWindow();
                return 0;
            }
