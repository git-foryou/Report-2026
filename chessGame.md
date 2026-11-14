## 피지컬 체스 게임 코드

      /* ======================================================================
       *  CHESS  -  C + raylib
       *
       *  Scenes : INTRO -> MENU -> GAME -> (GAME OVER) -> MENU / OUTRO
       *  Rules  : full legal-move generation, check / checkmate / stalemate,
       *           castling, en passant, promotion, 50-move rule,
       *           threefold repetition, insufficient material, time out
       *  Extras : 2-player mode, simple AI (negamax + alpha-beta, 3 levels),
       *           chess clock, undo, board flip, move hints, SAN move list,
       *           captured pieces + material count, pause / resign / draw
       *
       *  3D DUELS: ONLY when a capture happens (human or AI), the game switches to
       *           a first-person 3D arena. The attacker piece fights the defender
       *           piece (you vs. AI). Sword + gun, guard / PARRY, dash, shockwave,
       *           overdrive ultimate, enemy sword / gun / leap-slam attacks.
       *           Winner decides the capture; otherwise it is plain chess.
       *
       *  Build (Linux)  : gcc chess.c -o chess -lraylib -lm -lGL -lpthread -ldl -lrt -lX11
       *  Build (macOS)  : gcc chess.c -o chess -lraylib -framework IOKit -framework Cocoa -framework OpenGL
       *  Build (Windows): gcc chess.c -o chess.exe -lraylib -lopengl32 -lgdi32 -lwinmm
       *
       *  Board    : Mouse click = select / move
       *             U undo   F flip   H hints   R resign   D draw   ESC pause
       *  Duel     : WASD move, mouse look, LMB attack, RMB guard (tap just before
       *             a hit = PARRY, reflects bullets), Q dash, E shockwave,
       *             F overdrive, 1/2/Tab/Wheel switch weapon, R reload, ESC pause
       *
       *  NOTE: raylib's default font has no Hangul glyphs, so the UI is in English.
       *        (To use Korean, load a TTF with LoadFontEx and a codepoint table.)
       * ====================================================================== */
      #ifndef _CRT_SECURE_NO_WARNINGS
      #define _CRT_SECURE_NO_WARNINGS
      #endif
      #include "raylib.h"
      #include <stdio.h>
      #include <stdlib.h>
      #include <string.h>
      #include <math.h>
      #include <stdbool.h>
      
      #define SCREEN_W 1100
      #define SCREEN_H 760
      #define TILE 80
      #define BOARD_X 50
      #define BOARD_Y 60
      #define PANEL_X 730
      #define PANEL_W 340
      #define MAX_PLY 1024
      #define MAX_MOVES 320
      #define MATE_SCORE 100000
      
       /* NOTE: raylib defines WHITE / BLACK as colors, so sides are SIDE_W / SIDE_B */
      enum { PAWN = 1, KNIGHT, BISHOP, ROOK, QUEEN, KING };
      enum { SIDE_W = 1, SIDE_B = -1 };
      
      /* ---------------------------------------------------------------------
       *  Core chess data
       * ------------------------------------------------------------------- */
      typedef struct { int from, to, promo; } Move;
      
      typedef struct {
          signed char b[64];   /* index = row*8+col, row 0 = rank 8 (top). +white / -black */
          int turn;            /* SIDE_W or SIDE_B */
          int castle;          /* bit0 WK, bit1 WQ, bit2 BK, bit3 BQ */
          int ep;              /* en-passant target square or -1 */
          int halfmove;        /* 50-move counter */
          int fullmove;
      } Pos;
      
      static const int PVAL[7] = { 0, 100, 320, 330, 500, 900, 0 };
      
      static const int KN_DR[8] = { -2, -2, -1, -1, 1, 1, 2, 2 };
      static const int KN_DC[8] = { -1, 1, -2, 2, -2, 2, -1, 1 };
      static const int K_DR[8] = { -1, -1, -1, 0, 0, 1, 1, 1 };
      static const int K_DC[8] = { -1, 0, 1, -1, 1, -1, 0, 1 };
      static const int DG_DR[4] = { -1, -1, 1, 1 };
      static const int DG_DC[4] = { -1, 1, -1, 1 };
      static const int OR_DR[4] = { -1, 1, 0, 0 };
      static const int OR_DC[4] = { 0, 0, -1, 1 };
      
      static inline int RowOf(int s) { return s >> 3; }
      static inline int ColOf(int s) { return s & 7; }
      static int Sign(int v) { return v > 0 ? 1 : (v < 0 ? -1 : 0); }
      static int Dir(int side) { return side == SIDE_W ? -1 : 1; }
      static float ClampF(float v, float lo, float hi) { return v < lo ? lo : (v > hi ? hi : v); }
      
      static void InitPos(Pos* p)
      {
          static const int back[8] = { ROOK, KNIGHT, BISHOP, QUEEN, KING, BISHOP, KNIGHT, ROOK };
          memset(p, 0, sizeof * p);
          for (int c = 0; c < 8; c++) {
              p->b[c] = (signed char)-back[c];
              p->b[8 + c] = -PAWN;
              p->b[48 + c] = PAWN;
              p->b[56 + c] = (signed char)back[c];
          }
          p->turn = SIDE_W;
          p->castle = 15;
          p->ep = -1;
          p->halfmove = 0;
          p->fullmove = 1;
      }
      
      /* Is square `sq` attacked by side `by`? */
      static bool Attacked(const Pos* p, int sq, int by)
      {
          int r = RowOf(sq), c = ColOf(sq);
      
          int pr = r - Dir(by);
          for (int dc = -1; dc <= 1; dc += 2) {
              int pc = c + dc;
              if (pr >= 0 && pr < 8 && pc >= 0 && pc < 8 && p->b[pr * 8 + pc] == by * PAWN) return true;
          }
          for (int i = 0; i < 8; i++) {
              int nr = r + KN_DR[i], nc = c + KN_DC[i];
              if (nr >= 0 && nr < 8 && nc >= 0 && nc < 8 && p->b[nr * 8 + nc] == by * KNIGHT) return true;
          }
          for (int i = 0; i < 8; i++) {
              int nr = r + K_DR[i], nc = c + K_DC[i];
              if (nr >= 0 && nr < 8 && nc >= 0 && nc < 8 && p->b[nr * 8 + nc] == by * KING) return true;
          }
          for (int d = 0; d < 4; d++) {
              int nr = r + DG_DR[d], nc = c + DG_DC[d];
              while (nr >= 0 && nr < 8 && nc >= 0 && nc < 8) {
                  int v = p->b[nr * 8 + nc];
                  if (v != 0) { if (v == by * BISHOP || v == by * QUEEN) return true; break; }
                  nr += DG_DR[d]; nc += DG_DC[d];
              }
          }
          for (int d = 0; d < 4; d++) {
              int nr = r + OR_DR[d], nc = c + OR_DC[d];
              while (nr >= 0 && nr < 8 && nc >= 0 && nc < 8) {
                  int v = p->b[nr * 8 + nc];
                  if (v != 0) { if (v == by * ROOK || v == by * QUEEN) return true; break; }
                  nr += OR_DR[d]; nc += OR_DC[d];
              }
          }
          return false;
      }
      
      static int FindKing(const Pos* p, int side)
      {
          for (int s = 0; s < 64; s++) if (p->b[s] == side * KING) return s;
          return -1;
      }
      
      static bool InCheck(const Pos* p)
      {
          int k = FindKing(p, p->turn);
          return k >= 0 && Attacked(p, k, -p->turn);
      }
      
      static void AddMove(Move* l, int* n, int from, int to, int promo)
      {
          l[*n].from = from; l[*n].to = to; l[*n].promo = promo; (*n)++;
      }
      
      static void AddPawnMove(Move* l, int* n, int from, int to, bool promoting)
      {
          if (promoting) { for (int pt = QUEEN; pt >= KNIGHT; pt--) AddMove(l, n, from, to, pt); }
          else AddMove(l, n, from, to, 0);
      }
      
      /* Pseudo-legal moves (may leave own king in check) */
      static int GenPseudo(const Pos* p, Move* l)
      {
          int n = 0, side = p->turn;
          for (int s = 0; s < 64; s++) {
              int v = p->b[s];
              if (Sign(v) != side) continue;
              int t = abs(v), r = RowOf(s), c = ColOf(s);
      
              if (t == PAWN) {
                  int d = Dir(side);
                  int startRow = (side == SIDE_W) ? 6 : 1;
                  int promoRow = (side == SIDE_W) ? 0 : 7;
                  int nr = r + d;
                  if (nr < 0 || nr > 7) continue;
                  int to = nr * 8 + c;
                  if (p->b[to] == 0) {
                      AddPawnMove(l, &n, s, to, nr == promoRow);
                      if (r == startRow && p->b[(r + 2 * d) * 8 + c] == 0)
                          AddMove(l, &n, s, (r + 2 * d) * 8 + c, 0);
                  }
                  for (int dc = -1; dc <= 1; dc += 2) {
                      int nc = c + dc;
                      if (nc < 0 || nc > 7) continue;
                      int to2 = nr * 8 + nc;
                      if (Sign(p->b[to2]) == -side || (p->ep >= 0 && to2 == p->ep))
                          AddPawnMove(l, &n, s, to2, nr == promoRow);
                  }
              }
              else if (t == KNIGHT) {
                  for (int i = 0; i < 8; i++) {
                      int nr = r + KN_DR[i], nc = c + KN_DC[i];
                      if (nr < 0 || nr > 7 || nc < 0 || nc > 7) continue;
                      if (Sign(p->b[nr * 8 + nc]) != side) AddMove(l, &n, s, nr * 8 + nc, 0);
                  }
              }
              else if (t == KING) {
                  for (int i = 0; i < 8; i++) {
                      int nr = r + K_DR[i], nc = c + K_DC[i];
                      if (nr < 0 || nr > 7 || nc < 0 || nc > 7) continue;
                      if (Sign(p->b[nr * 8 + nc]) != side) AddMove(l, &n, s, nr * 8 + nc, 0);
                  }
                  /* castling */
                  int base = (side == SIDE_W) ? 60 : 4;
                  if (s == base && !Attacked(p, s, -side)) {
                      int kbit = (side == SIDE_W) ? 1 : 4;
                      int qbit = (side == SIDE_W) ? 2 : 8;
                      if ((p->castle & kbit) && p->b[base + 1] == 0 && p->b[base + 2] == 0 &&
                          p->b[base + 3] == side * ROOK &&
                          !Attacked(p, base + 1, -side) && !Attacked(p, base + 2, -side))
                          AddMove(l, &n, base, base + 2, 0);
                      if ((p->castle & qbit) && p->b[base - 1] == 0 && p->b[base - 2] == 0 &&
                          p->b[base - 3] == 0 && p->b[base - 4] == side * ROOK &&
                          !Attacked(p, base - 1, -side) && !Attacked(p, base - 2, -side))
                          AddMove(l, &n, base, base - 2, 0);
                  }
              }
              else {
                  for (int d = 0; d < 8; d++) {
                      bool diag = d < 4;
                      if (t == BISHOP && !diag) continue;
                      if (t == ROOK && diag) continue;
                      int dr = diag ? DG_DR[d] : OR_DR[d - 4];
                      int dc = diag ? DG_DC[d] : OR_DC[d - 4];
                      int nr = r + dr, nc = c + dc;
                      while (nr >= 0 && nr < 8 && nc >= 0 && nc < 8) {
                          int tv = p->b[nr * 8 + nc];
                          if (Sign(tv) == side) break;
                          AddMove(l, &n, s, nr * 8 + nc, 0);
                          if (tv != 0) break;
                          nr += dr; nc += dc;
                      }
                  }
              }
          }
          return n;
      }
      
      static int RightsMask(int sq)
      {
          switch (sq) {
          case 60: return ~3;
          case 63: return ~1;
          case 56: return ~2;
          case 4:  return ~12;
          case 7:  return ~4;
          case 0:  return ~8;
          default: return ~0;
          }
      }
      
      static void ApplyMove(Pos* p, Move m)
      {
          int piece = p->b[m.from];
          int t = abs(piece);
          int side = p->turn;
          bool capture = p->b[m.to] != 0;
      
          if (t == PAWN && ColOf(m.from) != ColOf(m.to) && p->b[m.to] == 0) {   /* en passant */
              p->b[RowOf(m.from) * 8 + ColOf(m.to)] = 0;
              capture = true;
          }
          if (t == KING && abs(m.to - m.from) == 2) {                             /* castling rook */
              if (m.to > m.from) { p->b[m.from + 1] = p->b[m.from + 3]; p->b[m.from + 3] = 0; }
              else { p->b[m.from - 1] = p->b[m.from - 4]; p->b[m.from - 4] = 0; }
          }
          p->b[m.to] = (signed char)piece;
          p->b[m.from] = 0;
          if (m.promo) p->b[m.to] = (signed char)(side * m.promo);
      
          p->castle &= RightsMask(m.from) & RightsMask(m.to);
          p->ep = -1;
          if (t == PAWN && abs(m.to - m.from) == 16) p->ep = (m.from + m.to) / 2;
          if (t == PAWN || capture) p->halfmove = 0; else p->halfmove++;
          if (side == SIDE_B) p->fullmove++;
          p->turn = -side;
      }
      
      static int GenLegal(const Pos* p, Move* out)
      {
          Move tmp[MAX_MOVES];
          int n = GenPseudo(p, tmp), k = 0;
          for (int i = 0; i < n; i++) {
              Pos q = *p;
              ApplyMove(&q, tmp[i]);
              int ks = FindKing(&q, p->turn);
              if (ks >= 0 && !Attacked(&q, ks, -p->turn)) out[k++] = tmp[i];
          }
          return k;
      }
      
      static bool InsufficientMaterial(const Pos* p)
      {
          int minors = 0, light = 0, dark = 0;
          for (int s = 0; s < 64; s++) {
              int t = abs(p->b[s]);
              if (t == PAWN || t == ROOK || t == QUEEN) return false;
              if (t == KNIGHT) minors++;
              else if (t == BISHOP) { minors++; if ((RowOf(s) + ColOf(s)) & 1) dark++; else light++; }
          }
          if (minors <= 1) return true;
          if (minors == light + dark && (light == 0 || dark == 0)) return true;
          return false;
      }
      
      static bool SamePos(const Pos* a, const Pos* b)
      {
          return a->turn == b->turn && a->castle == b->castle && a->ep == b->ep &&
              memcmp(a->b, b->b, 64) == 0;
      }
      
      /* Standard Algebraic Notation (with disambiguation and +/#) */
      static void MakeSAN(const Pos* p, Move m, char* out)
      {
          static const char* L = " PNBRQK";
          char buf[24];
          int k = 0;
          int piece = p->b[m.from], t = abs(piece);
          bool capture = p->b[m.to] != 0 || (t == PAWN && ColOf(m.from) != ColOf(m.to));
      
          if (t == KING && abs(m.to - m.from) == 2) {
              strcpy(buf, m.to > m.from ? "O-O" : "O-O-O");
              k = (int)strlen(buf);
          }
          else {
              if (t != PAWN) {
                  buf[k++] = L[t];
                  Move lg[MAX_MOVES];
                  int n = GenLegal(p, lg);
                  bool amb = false, sameFile = false, sameRank = false;
                  for (int i = 0; i < n; i++) {
                      if (lg[i].to == m.to && lg[i].from != m.from && p->b[lg[i].from] == piece) {
                          amb = true;
                          if (ColOf(lg[i].from) == ColOf(m.from)) sameFile = true;
                          if (RowOf(lg[i].from) == RowOf(m.from)) sameRank = true;
                      }
                  }
                  if (amb) {
                      if (!sameFile)      buf[k++] = (char)('a' + ColOf(m.from));
                      else if (!sameRank) buf[k++] = (char)('8' - RowOf(m.from));
                      else { buf[k++] = (char)('a' + ColOf(m.from)); buf[k++] = (char)('8' - RowOf(m.from)); }
                  }
              }
              else if (capture) {
                  buf[k++] = (char)('a' + ColOf(m.from));
              }
              if (capture) buf[k++] = 'x';
              buf[k++] = (char)('a' + ColOf(m.to));
              buf[k++] = (char)('8' - RowOf(m.to));
              if (m.promo) { buf[k++] = '='; buf[k++] = L[m.promo]; }
          }
          Pos q = *p;
          ApplyMove(&q, m);
          if (InCheck(&q)) {
              Move lg2[MAX_MOVES];
              buf[k++] = (GenLegal(&q, lg2) == 0) ? '#' : '+';
          }
          buf[k] = 0;
          strcpy(out, buf);
      }
      
      /* ---------------------------------------------------------------------
       *  AI : evaluation + negamax with alpha-beta
       * ------------------------------------------------------------------- */
      static int Eval(const Pos* p)   /* white's point of view, centipawns */
      {
          int s = 0;
          for (int sq = 0; sq < 64; sq++) {
              int v = p->b[sq];
              if (!v) continue;
              int side = Sign(v), t = abs(v), r = RowOf(sq), c = ColOf(sq);
              int val = PVAL[t];
              int cd = (abs(2 * c - 7) + abs(2 * r - 7)) / 2;      /* distance from center 1..7 */
              if (t == PAWN) {
                  int adv = (side == SIDE_W) ? (6 - r) : (r - 1);
                  val += adv * 6;
                  if (c == 3 || c == 4) val += 8;
              }
              else if (t == KNIGHT || t == BISHOP) {
                  val += (7 - cd) * 4;
              }
              else if (t == QUEEN) {
                  val += (7 - cd);
              }
              s += side * val;
          }
          return s;
      }
      
      static int Negamax(const Pos* p, int depth, int alpha, int beta, int ply)
      {
          if (depth == 0) return p->turn * Eval(p);
          Move mv[MAX_MOVES];
          int n = GenLegal(p, mv);
          if (n == 0) return InCheck(p) ? -MATE_SCORE + ply : 0;
          if (p->halfmove >= 100 || InsufficientMaterial(p)) return 0;
      
          int score[MAX_MOVES];
          for (int i = 0; i < n; i++) {
              int victim = abs(p->b[mv[i].to]), attacker = abs(p->b[mv[i].from]);
              score[i] = (victim ? 10 * PVAL[victim] - PVAL[attacker] : 0) + (mv[i].promo ? PVAL[mv[i].promo] : 0);
          }
          int best = -MATE_SCORE * 2;
          for (int i = 0; i < n; i++) {
              int bi = i;
              for (int j = i + 1; j < n; j++) if (score[j] > score[bi]) bi = j;
              Move tm = mv[i]; mv[i] = mv[bi]; mv[bi] = tm;
              int ts = score[i]; score[i] = score[bi]; score[bi] = ts;
      
              Pos q = *p;
              ApplyMove(&q, mv[i]);
              int v = -Negamax(&q, depth - 1, -beta, -alpha, ply + 1);
              if (v > best) best = v;
              if (best > alpha) alpha = best;
              if (alpha >= beta) break;
          }
          return best;
      }
      
      static Move ChooseAIMove(const Pos* p, int level)
      {
          Move mv[MAX_MOVES];
          int n = GenLegal(p, mv);
          int depth = level < 1 ? 1 : (level > 3 ? 3 : level);
          int jitter = (depth == 1) ? 40 : (depth == 2 ? 12 : 4);
          int bestI = 0, bestV = -MATE_SCORE * 4;
          for (int i = 0; i < n; i++) {
              Pos q = *p;
              ApplyMove(&q, mv[i]);
              int v = -Negamax(&q, depth - 1, -MATE_SCORE * 2, MATE_SCORE * 2, 1);
              v += GetRandomValue(0, jitter);
              if (v > bestV) { bestV = v; bestI = i; }
          }
          return mv[bestI];
      }
      
      /* ---------------------------------------------------------------------
       *  Game state
       * ------------------------------------------------------------------- */
      typedef enum { SCENE_INTRO, SCENE_MENU, SCENE_GAME, SCENE_DUEL, SCENE_OUTRO } Scene;
      typedef enum { MODAL_NONE, MODAL_PAUSE, MODAL_RESIGN, MODAL_DRAW, MODAL_PROMO } Modal;
      typedef enum { RES_NONE, RES_WHITE, RES_BLACK, RES_DRAW } Result;
      
      typedef struct {
          Pos hist[MAX_PLY + 1];       /* hist[i] = position after i plies */
          Move mv[MAX_PLY];
          char san[MAX_PLY][12];
          int nply;
      
          Move legal[MAX_MOVES];
          int nlegal;
          int selected;
      
          Modal modal;
          Move promoBase;
      
          Result result;
          char reason[64];
          bool hideOverlay;
      
          int mode;                    /* 0: 2 players, 1: you=White vs AI, 2: you=Black vs AI */
          int aiLevel;                 /* 1..3 */
          int tc;                      /* time-control index */
          bool flipped;
          bool hints;
          bool duels;                  /* 3D capture duels on/off */
          float clock[2];              /* [0]=white [1]=black seconds */
          float aiTimer;
      
          bool animOn;
          float animT;
          int animFrom, animTo, animPiece;
      
          int moveScroll;
          double startTime;
      
          char toast[64];
          float toastT;
      } Game;
      
      static Game G;
      static Scene gScene = SCENE_INTRO;
      static double gSceneStart = 0.0;
      static bool gQuit = false;
      static bool gClickUsed = false;
      static int gGames = 0, gWhiteWins = 0, gBlackWins = 0, gDraws = 0, gTotalPlies = 0;
      
      static const int TC_SEC[4] = { 0, 300, 600, 1800 };
      static const char* TC_NAME[4] = { "No limit", "5 min", "10 min", "30 min" };
      
      static void GoScene(Scene s) { gScene = s; gSceneStart = GetTime(); }
      static int AISide(void) { return G.mode == 1 ? SIDE_B : (G.mode == 2 ? SIDE_W : 0); }
      
      static void Toast(const char* msg)
      {
          snprintf(G.toast, sizeof G.toast, "%s", msg);
          G.toastT = 2.2f;
      }
      
      static void SetResult(Result r, const char* reason)
      {
          if (G.result != RES_NONE) return;
          G.result = r;
          snprintf(G.reason, sizeof G.reason, "%s", reason);
          G.hideOverlay = false;
          G.selected = -1;
          G.modal = MODAL_NONE;
          gGames++;
          gTotalPlies += G.nply;
          if (r == RES_WHITE) gWhiteWins++;
          else if (r == RES_BLACK) gBlackWins++;
          else gDraws++;
      }
      
      /* recompute legal moves of the current position and detect game end */
      static void RefreshLegal(void)
      {
          const Pos* cur = &G.hist[G.nply];
          G.nlegal = GenLegal(cur, G.legal);
          if (G.result != RES_NONE) return;
      
          if (FindKing(cur, SIDE_W) < 0) { SetResult(RES_BLACK, "King slain"); return; }
          if (FindKing(cur, SIDE_B) < 0) { SetResult(RES_WHITE, "King slain"); return; }
      
          if (G.nlegal == 0) {
              if (InCheck(cur)) SetResult(cur->turn == SIDE_W ? RES_BLACK : RES_WHITE, "Checkmate");
              else SetResult(RES_DRAW, "Stalemate");
          }
          else if (cur->halfmove >= 100) {
              SetResult(RES_DRAW, "50-move rule");
          }
          else if (InsufficientMaterial(cur)) {
              SetResult(RES_DRAW, "Insufficient material");
          }
          else {
              int rep = 0;
              for (int i = 0; i <= G.nply; i++) if (SamePos(&G.hist[i], cur)) rep++;
              if (rep >= 3) SetResult(RES_DRAW, "Threefold repetition");
          }
      }
      
      static void NewGame(void)
      {
          InitPos(&G.hist[0]);
          G.nply = 0;
          G.selected = -1;
          G.modal = MODAL_NONE;
          G.result = RES_NONE;
          G.reason[0] = 0;
          G.hideOverlay = false;
          G.clock[0] = G.clock[1] = (float)TC_SEC[G.tc];
          G.aiTimer = 0.0f;
          G.animOn = false;
          G.moveScroll = 0;
          G.startTime = GetTime();
          G.flipped = (G.mode == 2);
          G.toastT = 0.0f;
          RefreshLegal();
      }
      
      static void CommitMove(Move m)
      {
          if (G.nply >= MAX_PLY - 1) { SetResult(RES_DRAW, "Move limit reached"); return; }
          const Pos* cur = &G.hist[G.nply];
          MakeSAN(cur, m, G.san[G.nply]);
          G.mv[G.nply] = m;
          G.animOn = true; G.animT = 0.0f;
          G.animFrom = m.from; G.animTo = m.to; G.animPiece = cur->b[m.from];
          G.hist[G.nply + 1] = *cur;
          ApplyMove(&G.hist[G.nply + 1], m);
          G.nply++;
          G.selected = -1;
          G.moveScroll = 0;
          RefreshLegal();
      }
      
      static void DoUndo(void)
      {
          if (G.nply == 0 || G.result != RES_NONE) return;
          int ai = AISide(), steps = 1;
          if (ai != 0) {
              steps = (G.hist[G.nply].turn == -ai) ? 2 : 1;
              if (steps > G.nply) steps = G.nply;
          }
          G.nply -= steps;
          G.selected = -1;
          G.animOn = false;
          G.aiTimer = 0.0f;
          G.moveScroll = 0;
          RefreshLegal();
      }
      
      static void DoResign(void)
      {
          const Pos* cur = &G.hist[G.nply];
          int ai = AISide();
          int resigning = (ai != 0) ? -ai : cur->turn;
          SetResult(resigning == SIDE_W ? RES_BLACK : RES_WHITE, "Resignation");
      }
      
      static void AskDraw(void)
      {
          int ai = AISide();
          if (ai == 0) { G.modal = MODAL_DRAW; return; }
          int sc = Eval(&G.hist[G.nply]) * ai;
          if (sc <= 60) SetResult(RES_DRAW, "Draw by agreement");
          else Toast("The AI declined your draw offer.");
      }
      
      static int MaterialSum(const Pos* p, int side)
      {
          int m = 0;
          for (int s = 0; s < 64; s++) if (Sign(p->b[s]) == side) m += PVAL[abs(p->b[s])];
          return m;
      }
      
      static void CountMissing(const Pos* p, int side, int out[7])
      {
          static const int init[7] = { 0, 8, 2, 2, 2, 1, 1 };
          int cnt[7] = { 0 };
          for (int s = 0; s < 64; s++) if (Sign(p->b[s]) == side) cnt[abs(p->b[s])]++;
          for (int t = 0; t < 7; t++) { out[t] = init[t] - cnt[t]; if (out[t] < 0) out[t] = 0; }
          out[KING] = 0;
      }
      
      /* ---------------------------------------------------------------------
       *  Drawing helpers / UI
       * ------------------------------------------------------------------- */
      static void DrawTextC(const char* t, int cx, int y, int size, Color c)
      {
          DrawText(t, cx - MeasureText(t, size) / 2, y, size, c);
      }
      
      static bool UiButton(Rectangle r, const char* label, bool enabled, bool selected)
      {
          Vector2 m = GetMousePosition();
          bool hover = enabled && CheckCollisionPointRec(m, r);
          Color bg = !enabled ? (Color) { 62, 64, 72, 255 }
          : selected ? (Color) { 98, 150, 85, 255 }
          : hover ? (Color) { 100, 106, 136, 255 }
          : (Color) { 70, 76, 100, 255 };
          DrawRectangleRounded((Rectangle) { r.x, r.y + 3, r.width, r.height }, 0.25f, 8, (Color) { 0, 0, 0, 90 });
          DrawRectangleRounded(r, 0.25f, 8, bg);
          int fs = 20;
          Color tc = enabled ? RAYWHITE : (Color) { 130, 132, 140, 255 };
          DrawText(label, (int)(r.x + r.width / 2 - MeasureText(label, fs) / 2), (int)(r.y + r.height / 2 - fs / 2), fs, tc);
          if (hover && IsMouseButtonPressed(MOUSE_BUTTON_LEFT) && !gClickUsed) { gClickUsed = true; return true; }
          return false;
      }
      
      static void DrawDialog(Rectangle r, const char* title)
      {
          DrawRectangle(0, 0, SCREEN_W, SCREEN_H, (Color) { 0, 0, 0, 150 });
          DrawRectangleRounded((Rectangle) { r.x + 4, r.y + 6, r.width, r.height }, 0.06f, 8, (Color) { 0, 0, 0, 120 });
          DrawRectangleRounded(r, 0.06f, 8, (Color) { 38, 40, 54, 255 });
          DrawTextC(title, (int)(r.x + r.width / 2), (int)r.y + 18, 30, RAYWHITE);
      }
      
      /* soft radial glow (own helper: the built-in gradient-circle call changed signature between raylib versions) */
      static void DrawGlow(float cx, float cy, float rad, Color c)
      {
          for (int i = 0; i < 12; i++) {
              float k = (float)i / 12.0f;
              Color cc = c;
              cc.a = (unsigned char)(c.a * (k * k) / 6.0f);
              DrawCircleV((Vector2) { cx, cy }, rad* (1.0f - k), cc);
          }
      }
      
      static void DrawPieceGlyph(int piece, float cx, float cy, float rad, float alpha)
      {
          static const char* L[7] = { "", "P", "N", "B", "R", "Q", "K" };
          int t = abs(piece);
          if (t < 1 || t > 6) return;
          bool white = piece > 0;
          float al = ClampF(alpha, 0.0f, 1.0f);
          unsigned char a = (unsigned char)(255 * al);
          Color edge = white ? (Color) { 70, 60, 50, a } : (Color) { 225, 215, 200, a };
          Color fill = white ? (Color) { 252, 250, 244, a } : (Color) { 44, 44, 52, a };
          Color txt = white ? (Color) { 50, 44, 40, a } : (Color) { 240, 236, 228, a };
          DrawCircleV((Vector2) { cx + 2, cy + 3 }, rad, (Color) { 0, 0, 0, (unsigned char)(70 * al) });
          DrawCircleV((Vector2) { cx, cy }, rad, edge);
          DrawCircleV((Vector2) { cx, cy }, rad - rad * 0.08f - 1.0f, fill);
          int fs = (int)(rad * 1.1f);
          if (fs < 10) fs = 10;
          int w = MeasureText(L[t], fs);
          DrawText(L[t], (int)(cx - w / 2.0f), (int)(cy - fs / 2.0f), fs, txt);
      }
      
      static Vector2 SqToXY(int sq)
      {
          int r = RowOf(sq), c = ColOf(sq);
          if (G.flipped) { r = 7 - r; c = 7 - c; }
          return (Vector2) { (float)(BOARD_X + c * TILE), (float)(BOARD_Y + r * TILE) };
      }
      
      static int XYToSq(Vector2 m)
      {
          int c = (int)floorf((m.x - BOARD_X) / TILE);
          int r = (int)floorf((m.y - BOARD_Y) / TILE);
          if (c < 0 || c > 7 || r < 0 || r > 7) return -1;
          if (G.flipped) { r = 7 - r; c = 7 - c; }
          return r * 8 + c;
      }
      
      static const char* FmtClock(float s)
      {
          int t = (int)ceilf(s);
          if (t < 0) t = 0;
          return TextFormat("%02d:%02d", t / 60, t % 60);
      }
      
      static const char* ResultTitle(void)
      {
          int ai = AISide();
          switch (G.result) {
          case RES_WHITE: return ai ? (ai == SIDE_W ? "You lose..." : "You win!") : "White wins!";
          case RES_BLACK: return ai ? (ai == SIDE_B ? "You lose..." : "You win!") : "Black wins!";
          case RES_DRAW:  return "Draw";
          default:        return "";
          }
      }
      
      static void DrawBackdrop(double t)
      {
          ClearBackground((Color) { 24, 26, 36, 255 });
          int sz = 100;
          float off = fmodf((float)t * 18.0f, (float)(sz * 2));
          for (int r = -3; r < SCREEN_H / sz + 3; r++) {
              for (int c = -3; c < SCREEN_W / sz + 3; c++) {
                  if (((r + c) & 1) == 0) continue;
                  DrawRectangle(c * sz + (int)off, r * sz + (int)(off * 0.5f), sz, sz, (Color) { 34, 38, 54, 255 });
              }
          }
          DrawRectangleGradientV(0, 0, SCREEN_W, SCREEN_H, (Color) { 0, 0, 0, 30 }, (Color) { 0, 0, 0, 190 });
      }
      
      static void DrawFadeIn(double t, double dur)
      {
          float a = 1.0f - ClampF((float)(t / dur), 0.0f, 1.0f);
          if (a > 0.0f) DrawRectangle(0, 0, SCREEN_W, SCREEN_H, Fade(BLACK, a));
      }
      
      /* ---------------------------------------------------------------------
       *  INTRO
       * ------------------------------------------------------------------- */
      static void IntroFrame(double t)
      {
          DrawBackdrop(t);
      
          float ta = ClampF((float)((t - 0.3) / 0.8), 0.0f, 1.0f);
          DrawTextC("CHESS", SCREEN_W / 2 + 4, 154, 150, Fade(BLACK, 0.5f * ta));
          DrawTextC("CHESS", SCREEN_W / 2, 150, 150, Fade(RAYWHITE, ta));
      
          float sa = ClampF((float)((t - 1.0) / 0.8), 0.0f, 1.0f);
          DrawTextC("A game of kings - written in C with raylib", SCREEN_W / 2, 320, 26, Fade(GOLD, sa));
      
          static const int row[6] = { KING, QUEEN, ROOK, BISHOP, KNIGHT, PAWN };
          for (int i = 0; i < 6; i++) {
              float start = 1.2f + i * 0.15f;
              float k = ClampF((float)((t - start) / 0.5), 0.0f, 1.0f);
              float y = 470.0f - (1.0f - k) * (1.0f - k) * 300.0f;
              int pc = (i % 2 == 0) ? row[i] : -row[i];
              DrawPieceGlyph(pc, SCREEN_W / 2 + (i - 2.5f) * 120.0f, y, 44.0f, k);
          }
      
          if (t > 2.4) {
              float blink = 0.55f + 0.45f * sinf((float)t * 4.0f);
              DrawTextC("Press ENTER or click to start", SCREEN_W / 2, 620, 30, Fade(RAYWHITE, blink));
          }
          DrawFadeIn(t, 0.6);
      
          if (t > 0.6 && (IsKeyPressed(KEY_ENTER) || IsKeyPressed(KEY_SPACE) || IsMouseButtonPressed(MOUSE_BUTTON_LEFT)))
              GoScene(SCENE_MENU);
      }
      
      /* ---------------------------------------------------------------------
       *  MENU
       * ------------------------------------------------------------------- */
      static void MenuFrame(double t)
      {
          DrawBackdrop(t);
          DrawTextC("NEW GAME", SCREEN_W / 2, 28, 56, RAYWHITE);
      
          DrawTextC("Game mode", SCREEN_W / 2, 100, 22, LIGHTGRAY);
          const char* modes[3] = { "2 Players", "vs AI  (you: White)", "vs AI  (you: Black)" };
          for (int i = 0; i < 3; i++)
              if (UiButton((Rectangle) { (float)(110 + i * 300), 130, 280, 56 }, modes[i], true, G.mode == i)) G.mode = i;
      
          DrawTextC("AI strength  (also scales duel enemies)", SCREEN_W / 2, 205, 22, LIGHTGRAY);
          const char* lv[3] = { "Easy", "Normal", "Hard" };
          for (int i = 0; i < 3; i++)
              if (UiButton((Rectangle) { (float)(230 + i * 220), 235, 200, 46 }, lv[i], true, G.aiLevel == i + 1)) G.aiLevel = i + 1;
      
          DrawTextC("Time control (per player)", SCREEN_W / 2, 300, 22, LIGHTGRAY);
          for (int i = 0; i < 4; i++)
              if (UiButton((Rectangle) { (float)(220 + i * 170), 330, 150, 46 }, TC_NAME[i], true, G.tc == i)) G.tc = i;
      
          DrawTextC("Capture duels  (sword & gun fight in 3D when a piece is captured)", SCREEN_W / 2, 395, 22, LIGHTGRAY);
          if (UiButton((Rectangle) { SCREEN_W / 2 - 270.0f, 425, 260, 46 }, "Duels ON", true, G.duels)) G.duels = true;
          if (UiButton((Rectangle) { SCREEN_W / 2 + 10.0f, 425, 260, 46 }, "Duels OFF (plain chess)", true, !G.duels)) G.duels = false;
      
          if (UiButton((Rectangle) { SCREEN_W / 2 - 160, 500, 320, 74 }, "START GAME", true, false) ||
              IsKeyPressed(KEY_ENTER)) {
              NewGame();
              GoScene(SCENE_GAME);
              return;
          }
          if (UiButton((Rectangle) { SCREEN_W / 2 - 100, 592, 200, 46 }, "Quit", true, false)) {
              GoScene(SCENE_OUTRO);
              return;
          }
      
          DrawTextC("Board:  U undo   F flip   H hints   R resign   D draw   ESC pause",
              SCREEN_W / 2, 662, 20, GRAY);
          DrawTextC("Duel:  WASD + mouse  |  LMB attack  |  RMB guard / PARRY  |  Q dash  |  E shockwave  |  F overdrive",
              SCREEN_W / 2, 690, 20, GRAY);
          if (gGames > 0)
              DrawTextC(TextFormat("Session: %d games   W %d  /  B %d  /  Draw %d", gGames, gWhiteWins, gBlackWins, gDraws),
                  SCREEN_W / 2, 726, 18, (Color) { 140, 146, 170, 255 });
          DrawFadeIn(t, 0.4);
      }
      
      /* ---------------------------------------------------------------------
       *  3D DUEL  -  fired ONLY when a capture happens on the board
       *
       *  Attacker piece vs. defender piece fight in a first-person arena.
       *  You (WASD + mouse) fight an AI-controlled piece with sword & gun.
       *   - Winner of the duel decides the capture:
       *       attacker wins -> normal capture
       *       defender wins -> the attacker is slain, the defender stays
       * ------------------------------------------------------------------- */
      #define ARENA 11.5f
      #define EYE_H 1.7f
      #define PILLAR_R 0.9f
      #define MAX_BULLETS 32
      #define PI_F 3.14159265f
      
      typedef struct {
          int piece, side;
          float maxHp, swordMul, gunMul, speedMul, dashCd;
      } FStats;
      
      typedef struct {
          float x, y, z, vx, vy, vz, life, dmg;
          bool active, fromPlayer;
      } Bullet;
      
      typedef enum { E_IDLE, E_WINDUP_SWORD, E_WINDUP_SHOT, E_WINDUP_SLAM, E_RECOVER } EState;
      
      typedef struct {
          int phase;                   /* 0 intro, 1 fight, 2 result */
          float phaseT;
          bool paused;
          Move move;
          bool playerIsAttacker, playerWon, attackerWins;
          FStats pf, ef;
          float php, pmax, ehp, emax;
          float timeLeft;
          /* player */
          float px, pz, yaw, pitch;
          int weapon, ammo;                    /* weapon 0 = sword, 1 = gun */
          float atkCd, swingT, reloadT, fireT;
          bool swingHit, guarding;
          float parryT, parryCdT, dashT, dashCdT, shockCdT, iframes;
          float dvx, dvz;
          float ult, ultT;
          float hurtT, shake, sparkT, shockFx;
          int skipLook;
          /* enemy */
          float ex, ez, ey, eyaw;
          EState es;
          float esT, esDur;
          float estun, ehitT, strikeT, gunT;
          float eAtkCd, eShotCd, slamCd, strafeT, strafeDir;
          float slamX, slamZ, slamSX, slamSZ, slamFx;
          /* effects */
          Bullet bl[MAX_BULLETS];
          Vector3 trA, trB;
          float trT;
          char msg[32];
          float msgT;
      } Duel;
      
      static Duel D;
      static Camera3D gCam;
      static const char* PNAME[7] = { "", "Pawn", "Knight", "Bishop", "Rook", "Queen", "King" };
      static const float PILLARS[2][2] = { { -4.5f, 0.0f }, { 4.5f, 0.0f } };
      
      static FStats StatsFor(int piece, int side)
      {
          FStats s;
          s.piece = piece; s.side = side;
          switch (piece) {
          case PAWN:   s.maxHp = 70.0f;  s.swordMul = 1.0f; s.gunMul = 1.0f; s.speedMul = 1.00f; s.dashCd = 3.0f; break;
          case KNIGHT: s.maxHp = 95.0f;  s.swordMul = 1.0f; s.gunMul = 1.0f; s.speedMul = 1.12f; s.dashCd = 1.8f; break;
          case BISHOP: s.maxHp = 95.0f;  s.swordMul = 0.9f; s.gunMul = 1.4f; s.speedMul = 1.00f; s.dashCd = 3.0f; break;
          case ROOK:   s.maxHp = 135.0f; s.swordMul = 1.3f; s.gunMul = 0.9f; s.speedMul = 0.90f; s.dashCd = 3.5f; break;
          case QUEEN:  s.maxHp = 150.0f; s.swordMul = 1.2f; s.gunMul = 1.2f; s.speedMul = 1.10f; s.dashCd = 2.5f; break;
          default:     s.maxHp = 170.0f; s.swordMul = 1.1f; s.gunMul = 1.1f; s.speedMul = 1.00f; s.dashCd = 3.0f; break;
          }
          return s;
      }
      
      static Vector3 V3(float x, float y, float z) { Vector3 v; v.x = x; v.y = y; v.z = z; return v; }
      static float Dist2(float ax, float az, float bx, float bz) { float dx = ax - bx, dz = az - bz; return sqrtf(dx * dx + dz * dz); }
      
      static void DuelMsg(const char* m)
      {
          snprintf(D.msg, sizeof D.msg, "%s", m);
          D.msgT = 1.1f;
      }
      
      static Vector3 DuelForward3(void)
      {
          return V3(cosf(D.pitch) * sinf(D.yaw), sinf(D.pitch), cosf(D.pitch) * cosf(D.yaw));
      }
      
      /* keep a unit-circle body inside the arena and out of the pillars */
      static void Collide(float* x, float* z, float r)
      {
          float lim = ARENA - r;
          if (*x < -lim) *x = -lim;
          if (*x > lim) *x = lim;
          if (*z < -lim) *z = -lim;
          if (*z > lim) *z = lim;
          for (int i = 0; i < 2; i++) {
              float dx = *x - PILLARS[i][0], dz = *z - PILLARS[i][1];
              float d = sqrtf(dx * dx + dz * dz);
              if (d < PILLAR_R + r) {
                  if (d < 0.001f) { dx = 1.0f; dz = 0.0f; d = 1.0f; }
                  *x = PILLARS[i][0] + dx / d * (PILLAR_R + r);
                  *z = PILLARS[i][1] + dz / d * (PILLAR_R + r);
              }
          }
      }
      
      /* ray (normalized dir) vs sphere : returns distance or -1 */
      static float RaySphere(Vector3 o, Vector3 d, Vector3 c, float r)
      {
          float ox = o.x - c.x, oy = o.y - c.y, oz = o.z - c.z;
          float b = ox * d.x + oy * d.y + oz * d.z;
          float cc = ox * ox + oy * oy + oz * oz - r * r;
          float disc = b * b - cc;
          if (disc < 0.0f) return -1.0f;
          float s = sqrtf(disc);
          float t = -b - s;
          if (t < 0.0f) t = -b + s;
          return t;
      }
      
      /* ----- damage helpers ----- */
      static void HurtEnemy(float dmg)
      {
          if (D.estun > 0.0f) dmg *= 1.25f;
          D.ehp -= dmg;
          D.ehitT = 0.15f;
          if (D.ultT <= 0.0f) { D.ult += dmg * 0.8f; if (D.ult > 100.0f) D.ult = 100.0f; }
      }
      
      static void HurtPlayer(float dmg)
      {
          D.php -= dmg;
          D.hurtT = 0.35f;
          D.shake = 0.3f;
      }
      
      static void StunEnemy(float t)
      {
          D.estun = t; D.es = E_IDLE; D.esT = 0.0f; D.ey = 0.0f;
      }
      
      static void ParrySuccess(void)
      {
          StunEnemy(1.7f);
          HurtEnemy(10.0f);
          D.parryT = 0.0f; D.parryCdT = 0.25f;
          if (D.ultT <= 0.0f) { D.ult += 20.0f; if (D.ult > 100.0f) D.ult = 100.0f; }
          D.sparkT = 0.35f; D.shake = 0.3f;
          DuelMsg("PARRY!");
      }
      
      /* 0 = not guarded, 1 = blocked, 2 = perfect parry */
      static int GuardResult(float sx, float sz)
      {
          if (!D.guarding) return 0;
          float dx = sx - D.px, dz = sz - D.pz;
          float L = sqrtf(dx * dx + dz * dz);
          if (L < 0.001f) return 1;
          float fx = sinf(D.yaw), fz = cosf(D.yaw);
          if ((fx * dx + fz * dz) / L < 0.0f) return 0;
          return D.parryT > 0.0f ? 2 : 1;
      }
      
      static void FireEnemyBullet(int lvl)
      {
          for (int i = 0; i < MAX_BULLETS; i++) {
              if (D.bl[i].active) continue;
              float fx = sinf(D.eyaw), fz = cosf(D.eyaw);
              float sx = D.ex + fx * 0.7f, sy = 1.35f, sz = D.ez + fz * 0.7f;
              float spread = (lvl == 1 ? 0.9f : (lvl == 2 ? 0.5f : 0.2f));
              float tx = D.px + (GetRandomValue(-100, 100) / 100.0f) * spread;
              float tz = D.pz + (GetRandomValue(-100, 100) / 100.0f) * spread;
              float ty = 1.2f;
              float dx = tx - sx, dy = ty - sy, dz = tz - sz;
              float L = sqrtf(dx * dx + dy * dy + dz * dz);
              float spd = 15.0f + lvl * 2.0f;
              Bullet* b = &D.bl[i];
              b->active = true; b->fromPlayer = false;
              b->x = sx; b->y = sy; b->z = sz;
              b->vx = dx / L * spd; b->vy = dy / L * spd; b->vz = dz / L * spd;
              b->life = 3.0f;
              b->dmg = (7.0f + lvl * 2.5f) * D.ef.gunMul;
              break;
          }
          D.gunT = 0.5f;
      }
      
      /* ----- enemy AI ----- */
      static void EnemyUpdate(float dt)
      {
          int lvl = G.aiLevel;
          float windup = lvl == 1 ? 0.70f : (lvl == 2 ? 0.52f : 0.40f);
          float dmgBase = lvl == 1 ? 11.0f : (lvl == 2 ? 15.0f : 19.0f);
          float spd = (lvl == 1 ? 3.0f : (lvl == 2 ? 3.6f : 4.2f)) * D.ef.speedMul;
          float dx = D.px - D.ex, dz = D.pz - D.ez;
          float d = sqrtf(dx * dx + dz * dz);
          if (d < 0.001f) d = 0.001f;
      
          D.ehitT -= dt; D.strikeT -= dt; D.slamFx -= dt; D.gunT -= dt;
          D.eAtkCd -= dt; D.eShotCd -= dt; D.slamCd -= dt;
          D.eyaw = atan2f(dx, dz);
      
          if (D.estun > 0.0f) { D.estun -= dt; D.ey = 0.0f; return; }
      
          switch (D.es) {
          case E_IDLE: {
              D.strafeT -= dt;
              if (D.strafeT <= 0.0f) {
                  D.strafeDir = GetRandomValue(0, 1) ? 1.0f : -1.0f;
                  D.strafeT = 1.0f + GetRandomValue(0, 15) / 10.0f;
              }
              float mx = (-dz / d) * D.strafeDir * 0.7f, mz = (dx / d) * D.strafeDir * 0.7f;
              if (d > 3.0f) { mx += dx / d; mz += dz / d; }
              else if (d < 1.6f) { mx -= dx / d * 0.6f; mz -= dz / d * 0.6f; }
              float ml = sqrtf(mx * mx + mz * mz);
              if (ml > 0.001f) { D.ex += mx / ml * spd * dt; D.ez += mz / ml * spd * dt; }
              Collide(&D.ex, &D.ez, 0.5f);
      
              if (d < 2.7f && D.eAtkCd <= 0.0f) {
                  D.es = E_WINDUP_SWORD; D.esDur = D.esT = windup;
              }
              else if (d > 5.0f && D.slamCd <= 0.0f && GetRandomValue(0, 100) < 3) {
                  D.es = E_WINDUP_SLAM; D.esDur = D.esT = 1.15f;
                  D.slamX = D.px; D.slamZ = D.pz; D.slamSX = D.ex; D.slamSZ = D.ez;
                  D.slamCd = lvl == 1 ? 11.0f : (lvl == 2 ? 9.0f : 7.0f);
              }
              else if (d > 4.5f && D.eShotCd <= 0.0f && GetRandomValue(0, 100) < 4) {
                  D.es = E_WINDUP_SHOT; D.esDur = D.esT = 0.55f;
              }
          } break;
      
          case E_WINDUP_SWORD:
              D.esT -= dt;
              if (d > 1.7f) {                                   /* lunge */
                  D.ex += dx / d * 2.2f * dt; D.ez += dz / d * 2.2f * dt;
                  Collide(&D.ex, &D.ez, 0.5f);
              }
              if (D.esT <= 0.0f) {
                  float dmg = dmgBase * D.ef.swordMul;
                  D.strikeT = 0.25f;
                  if (d < 3.0f && D.iframes <= 0.0f) {
                      int g = GuardResult(D.ex, D.ez);
                      if (g == 2) ParrySuccess();
                      else {
                          if (g == 1) { dmg *= 0.25f; DuelMsg("BLOCK"); if (D.ultT <= 0.0f) { D.ult += 3.0f; } }
                          HurtPlayer(dmg);
                      }
                  }
                  D.es = E_RECOVER; D.esDur = D.esT = 0.5f;
                  D.eAtkCd = lvl == 1 ? 1.5f : (lvl == 2 ? 1.0f : 0.7f);
              }
              break;
      
          case E_WINDUP_SHOT:
              D.esT -= dt;
              if (D.esT <= 0.0f) {
                  FireEnemyBullet(lvl);
                  D.es = E_RECOVER; D.esDur = D.esT = 0.45f;
                  D.eShotCd = lvl == 1 ? 3.0f : (lvl == 2 ? 2.2f : 1.5f);
              }
              break;
      
          case E_WINDUP_SLAM: {
              D.esT -= dt;
              float p = 1.0f - D.esT / D.esDur;
              if (p > 1.0f) p = 1.0f;
              D.ex = D.slamSX + (D.slamX - D.slamSX) * p;
              D.ez = D.slamSZ + (D.slamZ - D.slamSZ) * p;
              D.ey = sinf(PI_F * p) * 2.6f;
              if (D.esT <= 0.0f) {
                  D.ey = 0.0f;
                  Collide(&D.ex, &D.ez, 0.5f);
                  D.slamFx = 0.45f; D.shake = 0.35f;
                  if (Dist2(D.px, D.pz, D.slamX, D.slamZ) < 3.4f && D.iframes <= 0.0f)
                      HurtPlayer((18.0f + lvl * 4.0f) * D.ef.swordMul);
                  D.es = E_RECOVER; D.esDur = D.esT = 1.0f;
              }
          } break;
      
          case E_RECOVER:
              D.esT -= dt;
              D.ey = 0.0f;
              if (D.esT <= 0.0f) D.es = E_IDLE;
              break;
          }
      }
      
      /* ----- player ----- */
      static void PlayerLook(void)
      {
          if (D.skipLook > 0) { D.skipLook--; return; }
          Vector2 md = GetMouseDelta();
          D.yaw -= md.x * 0.0028f;
          D.pitch -= md.y * 0.0028f;
          D.pitch = ClampF(D.pitch, -1.45f, 1.45f);
      }
      
      static void PlayerSwordResolve(void)
      {
          float dx = D.ex - D.px, dz = D.ez - D.pz;
          float d = sqrtf(dx * dx + dz * dz);
          if (d < 0.001f) d = 0.001f;
          float fx = sinf(D.yaw), fz = cosf(D.yaw);
          float dot = (fx * dx + fz * dz) / d;
          if (d < 2.9f && dot > 0.35f && D.ey < 1.0f) {
              float dmg = 24.0f * D.pf.swordMul * (D.ultT > 0.0f ? 1.5f : 1.0f);
              HurtEnemy(dmg);
              D.ex += dx / d * 0.7f; D.ez += dz / d * 0.7f;
              Collide(&D.ex, &D.ez, 0.5f);
              D.sparkT = 0.12f;
          }
      }
      
      static void PlayerGunFire(void)
      {
          if (D.ammo <= 0) { D.reloadT = 1.2f; return; }
          D.ammo--;
          D.atkCd = D.ultT > 0.0f ? 0.13f : 0.17f;
          D.fireT = 0.1f;
      
          Vector3 o = V3(D.px, EYE_H, D.pz), dir = DuelForward3();
          float tBody = RaySphere(o, dir, V3(D.ex, 1.0f + D.ey, D.ez), 0.62f);
          float tHead = RaySphere(o, dir, V3(D.ex, 1.8f + D.ey, D.ez), 0.30f);
          float tWall = -1.0f;
          for (int i = 0; i < 2; i++)
              for (int k = 0; k < 3; k++) {
                  float t = RaySphere(o, dir, V3(PILLARS[i][0], 0.5f + k * 0.9f, PILLARS[i][1]), PILLAR_R);
                  if (t > 0.0f && (tWall < 0.0f || t < tWall)) tWall = t;
              }
          float best = -1.0f;
          bool head = false;
          if (tHead > 0.0f) { best = tHead; head = true; }
          if (tBody > 0.0f && (best < 0.0f || tBody < best)) { best = tBody; head = false; }
      
          Vector3 end;
          if (best > 0.0f && (tWall < 0.0f || best < tWall)) {
              float dmg = (head ? 26.0f : 13.0f) * D.pf.gunMul * (D.ultT > 0.0f ? 1.5f : 1.0f);
              HurtEnemy(dmg);
              if (head) DuelMsg("HEADSHOT!");
              end = V3(o.x + dir.x * best, o.y + dir.y * best, o.z + dir.z * best);
          }
          else {
              float t = tWall > 0.0f ? tWall : 40.0f;
              end = V3(o.x + dir.x * t, o.y + dir.y * t, o.z + dir.z * t);
          }
          float rx = -cosf(D.yaw), rz = sinf(D.yaw);
          D.trA = V3(o.x + rx * 0.3f + dir.x * 0.8f, o.y - 0.25f + dir.y * 0.8f, o.z + rz * 0.3f + dir.z * 0.8f);
          D.trB = end;
          D.trT = 0.07f;
      }
      
      static void PlayerUpdate(float dt)
      {
          PlayerLook();
      
          D.atkCd -= dt; D.dashCdT -= dt; D.shockCdT -= dt; D.parryCdT -= dt;
          D.parryT -= dt; D.iframes -= dt; D.swingT -= dt;
          if (D.ultT > 0.0f) { D.ultT -= dt; D.php += 4.0f * dt; if (D.php > D.pmax) D.php = D.pmax; }
          if (D.reloadT > 0.0f) { D.reloadT -= dt; if (D.reloadT <= 0.0f) D.ammo = 12; }
      
          /* guard / parry */
          bool rmb = IsMouseButtonDown(MOUSE_BUTTON_RIGHT);
          if (rmb && !D.guarding) {
              D.guarding = true;
              if (D.parryCdT <= 0.0f) { D.parryT = 0.30f; D.parryCdT = 0.85f; }
          }
          if (!rmb) D.guarding = false;
      
          /* movement */
          float fx = sinf(D.yaw), fz = cosf(D.yaw);
          float rx = -cosf(D.yaw), rz = sinf(D.yaw);
          float mx = 0.0f, mz = 0.0f;
          if (IsKeyDown(KEY_W)) { mx += fx; mz += fz; }
          if (IsKeyDown(KEY_S)) { mx -= fx; mz -= fz; }
          if (IsKeyDown(KEY_D)) { mx += rx; mz += rz; }
          if (IsKeyDown(KEY_A)) { mx -= rx; mz -= rz; }
          float ml = sqrtf(mx * mx + mz * mz);
          if (ml > 0.0f) { mx /= ml; mz /= ml; }
          float spd = 6.0f * D.pf.speedMul * (D.guarding ? 0.45f : 1.0f) * (D.ultT > 0.0f ? 1.2f : 1.0f);
          if (D.dashT > 0.0f) { D.dashT -= dt; D.px += D.dvx * dt; D.pz += D.dvz * dt; }
          else { D.px += mx * spd * dt; D.pz += mz * spd * dt; }
          Collide(&D.px, &D.pz, 0.45f);
          {
              float dx = D.px - D.ex, dz = D.pz - D.ez;
              float d = sqrtf(dx * dx + dz * dz);
              if (d < 0.95f && d > 0.001f) { D.px = D.ex + dx / d * 0.95f; D.pz = D.ez + dz / d * 0.95f; Collide(&D.px, &D.pz, 0.45f); }
          }
      
          /* abilities */
          if (IsKeyPressed(KEY_Q) && D.dashCdT <= 0.0f) {
              float ddx = ml > 0.0f ? mx : fx, ddz = ml > 0.0f ? mz : fz;
              D.dvx = ddx * 24.0f; D.dvz = ddz * 24.0f;
              D.dashT = 0.17f; D.iframes = 0.25f; D.dashCdT = D.pf.dashCd;
          }
          if (IsKeyPressed(KEY_E) && D.shockCdT <= 0.0f) {
              D.shockCdT = 8.0f; D.shockFx = 0.45f; D.shake = 0.3f;
              float dx = D.ex - D.px, dz = D.ez - D.pz;
              float d = sqrtf(dx * dx + dz * dz);
              if (d < 4.6f) {
                  if (d < 0.001f) d = 0.001f;
                  HurtEnemy(14.0f);
                  StunEnemy(1.3f);
                  D.ex += dx / d * 2.5f; D.ez += dz / d * 2.5f;
                  Collide(&D.ex, &D.ez, 0.5f);
                  DuelMsg("SHOCKWAVE!");
              }
              for (int i = 0; i < MAX_BULLETS; i++)
                  if (D.bl[i].active && !D.bl[i].fromPlayer && Dist2(D.bl[i].x, D.bl[i].z, D.px, D.pz) < 4.6f)
                      D.bl[i].active = false;
          }
          if (IsKeyPressed(KEY_F) && D.ult >= 100.0f && D.ultT <= 0.0f) {
              D.ultT = 7.0f; D.ult = 0.0f; DuelMsg("OVERDRIVE!");
          }
          if (IsKeyPressed(KEY_R) && D.weapon == 1 && D.ammo < 12 && D.reloadT <= 0.0f) D.reloadT = 1.2f;
      
          /* weapon switch */
          int prevW = D.weapon;
          if (IsKeyPressed(KEY_ONE)) D.weapon = 0;
          if (IsKeyPressed(KEY_TWO)) D.weapon = 1;
          if (IsKeyPressed(KEY_TAB) || GetMouseWheelMove() != 0.0f) D.weapon = 1 - D.weapon;
          if (D.weapon != prevW) { D.atkCd = 0.2f; D.reloadT = 0.0f; }
      
          /* attack */
          if (IsMouseButtonDown(MOUSE_BUTTON_LEFT) && !D.guarding && D.atkCd <= 0.0f) {
              if (D.weapon == 0) {
                  D.swingT = 0.38f; D.swingHit = false;
                  D.atkCd = D.ultT > 0.0f ? 0.40f : 0.55f;
              }
              else if (D.reloadT <= 0.0f) {
                  PlayerGunFire();
              }
          }
          if (D.swingT > 0.0f && !D.swingHit && D.swingT < 0.22f) { D.swingHit = true; PlayerSwordResolve(); }
      }
      
      static void BulletsUpdate(float dt)
      {
          for (int i = 0; i < MAX_BULLETS; i++) {
              Bullet* b = &D.bl[i];
              if (!b->active) continue;
              b->x += b->vx * dt; b->y += b->vy * dt; b->z += b->vz * dt;
              b->life -= dt;
              if (b->life <= 0.0f || fabsf(b->x) > ARENA + 0.5f || fabsf(b->z) > ARENA + 0.5f || b->y < 0.0f || b->y > 4.0f) { b->active = false; continue; }
              for (int k = 0; k < 2; k++)
                  if (Dist2(b->x, b->z, PILLARS[k][0], PILLARS[k][1]) < PILLAR_R && b->y < 3.0f) b->active = false;
              if (!b->active) continue;
      
              if (!b->fromPlayer) {
                  if (Dist2(b->x, b->z, D.px, D.pz) < 0.55f && b->y > 0.1f && b->y < 1.95f && D.iframes <= 0.0f) {
                      int g = GuardResult(b->x - b->vx * 0.1f, b->z - b->vz * 0.1f);
                      if (g == 2) {                                         /* reflect */
                          float dx = D.ex - b->x, dy = (1.1f + D.ey) - b->y, dz = D.ez - b->z;
                          float L = sqrtf(dx * dx + dy * dy + dz * dz);
                          if (L < 0.001f) L = 0.001f;
                          b->vx = dx / L * 24.0f; b->vy = dy / L * 24.0f; b->vz = dz / L * 24.0f;
                          b->fromPlayer = true; b->life = 2.0f; b->dmg = 20.0f;
                          D.parryT = 0.0f; D.parryCdT = 0.25f; D.sparkT = 0.3f; D.shake = 0.2f;
                          DuelMsg("REFLECTED!");
                      }
                      else {
                          float dmg = b->dmg * (g == 1 ? 0.2f : 1.0f);
                          if (g == 1) DuelMsg("BLOCK");
                          HurtPlayer(dmg);
                          b->active = false;
                      }
                  }
              }
              else {
                  if (Dist2(b->x, b->z, D.ex, D.ez) < 0.75f && b->y < 2.2f + D.ey) {
                      HurtEnemy(b->dmg);
                      b->active = false;
                  }
              }
          }
      }
      
      /* ----- start / finish ----- */
      static void StartDuel(Move m)
      {
          const Pos* cur = &G.hist[G.nply];
          int att = cur->b[m.from];
          int def = cur->b[m.to];
          if (def == 0) def = cur->b[RowOf(m.from) * 8 + ColOf(m.to)];       /* en passant */
          int ai = AISide();
      
          memset(&D, 0, sizeof D);
          D.move = m;
          D.playerIsAttacker = (ai == 0) ? true : (Sign(att) != ai);
          int pp = D.playerIsAttacker ? att : def;
          int ee = D.playerIsAttacker ? def : att;
          D.pf = StatsFor(abs(pp), Sign(pp));
          D.ef = StatsFor(abs(ee), Sign(ee));
          float hpMul = G.aiLevel == 1 ? 0.8f : (G.aiLevel == 2 ? 1.0f : 1.25f);
          D.pmax = D.php = D.pf.maxHp;
          D.emax = D.ehp = D.ef.maxHp * hpMul;
      
          D.px = 0.0f; D.pz = -8.0f; D.yaw = 0.0f; D.pitch = 0.0f;
          D.ex = 0.0f; D.ez = 8.0f; D.eyaw = PI_F;
          D.ammo = 12;
          D.timeLeft = 90.0f;
          D.eAtkCd = 1.5f; D.eShotCd = 2.0f; D.slamCd = 5.0f; D.strafeDir = 1.0f; D.strafeT = 1.0f;
          D.skipLook = 3;
      
          G.modal = MODAL_NONE;
          G.selected = -1;
          DisableCursor();
          GoScene(SCENE_DUEL);
      }
      
      static void CommitDuelLoss(Move m)
      {
          if (G.nply >= MAX_PLY - 1) { SetResult(RES_DRAW, "Move limit reached"); return; }
          const Pos* cur = &G.hist[G.nply];
          char base[16];
          MakeSAN(cur, m, base);
          size_t L = strlen(base);
          while (L > 0 && (base[L - 1] == '+' || base[L - 1] == '#')) base[--L] = 0;
          base[8] = 0;
          snprintf(G.san[G.nply], sizeof G.san[G.nply], "%s(L)", base);
          G.mv[G.nply] = m;
      
          Pos q = *cur;                       /* the attacker is slain: it simply vanishes */
          q.b[m.from] = 0;
          q.castle &= RightsMask(m.from);
          q.ep = -1;
          q.halfmove = 0;
          if (cur->turn == SIDE_B) q.fullmove++;
          q.turn = -cur->turn;
          G.hist[G.nply + 1] = q;
          G.nply++;
          G.selected = -1;
          G.moveScroll = 0;
          G.animOn = false;
          RefreshLegal();
      }
      
      static void FinishDuel(void)
      {
          EnableCursor();
          Move m = D.move;
          bool win = D.attackerWins;
          GoScene(SCENE_GAME);
          G.aiTimer = 0.0f;
          if (win) CommitMove(m); else CommitDuelLoss(m);
      }
      
      /* every move goes through here: a capture triggers the 3D duel, otherwise plain chess */
      static void TryMove(Move m)
      {
          const Pos* cur = &G.hist[G.nply];
          int victim = cur->b[m.to];
          if (abs(cur->b[m.from]) == PAWN && ColOf(m.from) != ColOf(m.to) && victim == 0)
              victim = cur->b[RowOf(m.from) * 8 + ColOf(m.to)];                /* en passant */
          if (G.duels && victim != 0 && abs(victim) != KING) StartDuel(m);
          else CommitMove(m);
      }
      
      static void EndDuel(bool playerWon)
      {
          D.phase = 2; D.phaseT = 0.0f;
          D.playerWon = playerWon;
          D.attackerWins = (playerWon == D.playerIsAttacker);
          D.guarding = false;
      }
      
      /* returns true when the duel is over and the scene has changed */
      static bool UpdateDuel(float dt)
      {
          D.msgT -= dt; D.hurtT -= dt; D.shake -= dt * 1.5f; D.sparkT -= dt;
          D.shockFx -= dt; D.trT -= dt; D.fireT -= dt;
      
          if (D.phase == 0) {
              PlayerLook();
              if (D.phaseT >= 2.8f) { D.phase = 1; D.phaseT = 0.0f; }
              return false;
          }
          if (D.phase == 2) {
              D.ehitT -= dt;
              if ((D.phaseT > 1.3f && (IsKeyPressed(KEY_ENTER) || IsKeyPressed(KEY_SPACE) || IsMouseButtonPressed(MOUSE_BUTTON_LEFT))) ||
                  D.phaseT > 4.5f) {
                  FinishDuel();
                  return true;
              }
              return false;
          }
          D.timeLeft -= dt;
          PlayerUpdate(dt);
          EnemyUpdate(dt);
          BulletsUpdate(dt);
      
          if (D.php <= 0.0f) { D.php = 0.0f; EndDuel(false); }
          else if (D.ehp <= 0.0f) { D.ehp = 0.0f; EndDuel(true); }
          else if (D.timeLeft <= 0.0f) { D.timeLeft = 0.0f; EndDuel(D.php / D.pmax > D.ehp / D.emax); }
          return false;
      }
      
      /* ----- 3D drawing ----- */
      static void UpdateCam(void)
      {
          Vector3 f = DuelForward3();
          float sx = 0.0f, sy = 0.0f, sz = 0.0f;
          if (D.shake > 0.0f) {
              sx = GetRandomValue(-100, 100) / 100.0f * D.shake * 0.1f;
              sy = GetRandomValue(-100, 100) / 100.0f * D.shake * 0.1f;
              sz = GetRandomValue(-100, 100) / 100.0f * D.shake * 0.1f;
          }
          gCam.position = V3(D.px + sx, EYE_H + sy, D.pz + sz);
          gCam.target = V3(gCam.position.x + f.x, gCam.position.y + f.y, gCam.position.z + f.z);
          gCam.up = V3(0.0f, 1.0f, 0.0f);
          gCam.fovy = 75.0f + (D.dashT > 0.0f ? 10.0f : 0.0f);
          gCam.projection = CAMERA_PERSPECTIVE;
      }
      
      static void DrawEnemy(void)
      {
          static const float SC[7] = { 1.0f, 0.92f, 1.0f, 1.03f, 1.12f, 1.08f, 1.18f };
          float sc = SC[D.ef.piece];
          float x = D.ex, y = D.ey, z = D.ez;
          float fx = sinf(D.eyaw), fz = cosf(D.eyaw);
          float rx = -cosf(D.eyaw), rz = sinf(D.eyaw);
          bool white = D.ef.side > 0;
          Color armor = white ? (Color) { 226, 226, 232, 255 } : (Color) { 58, 58, 70, 255 };
          Color trim = white ? (Color) { 200, 165, 60, 255 } : (Color) { 170, 30, 44, 255 };
          Color dark = white ? (Color) { 150, 150, 160, 255 } : (Color) { 30, 30, 38, 255 };
          if (D.ehitT > 0.0f) armor = (Color){ 255, 120, 110, 255 };
          if (D.estun > 0.0f && (((int)(GetTime() * 12.0)) & 1)) armor = (Color){ 130, 170, 255, 255 };
      
          DrawCylinder(V3(x, 0.01f, z), 0.55f, 0.55f, 0.02f, 16, (Color) { 0, 0, 0, 90 });
      
          /* legs */
          float sw = sinf((float)GetTime() * 7.0f) * ((D.es == E_IDLE && D.estun <= 0.0f) ? 0.18f : 0.0f);
          DrawCylinderEx(V3(x + rx * 0.17f + fx * sw, y, z + rz * 0.17f + fz * sw),
              V3(x + rx * 0.17f, y + 0.9f * sc, z + rz * 0.17f), 0.11f, 0.14f, 8, dark);
          DrawCylinderEx(V3(x - rx * 0.17f - fx * sw, y, z - rz * 0.17f - fz * sw),
              V3(x - rx * 0.17f, y + 0.9f * sc, z - rz * 0.17f), 0.11f, 0.14f, 8, dark);
          /* torso */
          DrawCylinderEx(V3(x, y + 0.85f * sc, z), V3(x, y + 1.55f * sc, z), 0.30f * sc, 0.36f * sc, 12, armor);
          DrawCylinderEx(V3(x, y + 0.85f * sc, z), V3(x, y + 0.98f * sc, z), 0.31f * sc, 0.31f * sc, 12, trim);
          /* head */
          Vector3 head = V3(x, y + 1.80f * sc, z);
          DrawSphere(head, 0.21f, armor);
          DrawSphere(V3(head.x + fx * 0.14f, head.y, head.z + fz * 0.14f), 0.10f, dark);
          switch (D.ef.piece) {
          case KNIGHT:
              DrawCylinderEx(V3(head.x + rx * 0.12f, head.y + 0.16f, head.z + rz * 0.12f), V3(head.x + rx * 0.14f, head.y + 0.46f, head.z + rz * 0.14f), 0.07f, 0.0f, 6, trim);
              DrawCylinderEx(V3(head.x - rx * 0.12f, head.y + 0.16f, head.z - rz * 0.12f), V3(head.x - rx * 0.14f, head.y + 0.46f, head.z - rz * 0.14f), 0.07f, 0.0f, 6, trim);
              break;
          case BISHOP:
              DrawCylinderEx(V3(head.x, head.y + 0.15f, head.z), V3(head.x, head.y + 0.62f, head.z), 0.19f, 0.0f, 10, trim);
              break;
          case ROOK:
              DrawCylinderEx(V3(head.x, head.y + 0.12f, head.z), V3(head.x, head.y + 0.36f, head.z), 0.25f, 0.25f, 8, trim);
              break;
          case QUEEN:
              for (int i = 0; i < 5; i++) {
                  float a = i * 2.0f * PI_F / 5.0f;
                  DrawSphere(V3(head.x + cosf(a) * 0.17f, head.y + 0.28f, head.z + sinf(a) * 0.17f), 0.07f, trim);
              }
              break;
          case KING:
              DrawCylinderEx(V3(head.x, head.y + 0.18f, head.z), V3(head.x, head.y + 0.62f, head.z), 0.05f, 0.05f, 6, trim);
              DrawCylinderEx(V3(head.x - rx * 0.16f, head.y + 0.46f, head.z - rz * 0.16f), V3(head.x + rx * 0.16f, head.y + 0.46f, head.z + rz * 0.16f), 0.05f, 0.05f, 6, trim);
              break;
          default: break;
          }
      
          /* arms */
          Vector3 sL = V3(x - rx * 0.40f * sc, y + 1.45f * sc, z - rz * 0.40f * sc);
          Vector3 sR = V3(x + rx * 0.40f * sc, y + 1.45f * sc, z + rz * 0.40f * sc);
          Vector3 handL = V3(sL.x + fx * 0.15f, sL.y - 0.55f, sL.z + fz * 0.15f);
          DrawCylinderEx(sL, handL, 0.08f, 0.07f, 8, armor);
      
          Vector3 hand, tip;
          bool gun = (D.es == E_WINDUP_SHOT) || D.gunT > 0.0f;
          float p = (D.esDur > 0.0f) ? 1.0f - D.esT / D.esDur : 0.0f;
          if (p < 0.0f) p = 0.0f;
          if (p > 1.0f) p = 1.0f;
          Color blade = (Color){ 225, 228, 238, 255 };
          if (gun) {
              hand = V3(sR.x + fx * 0.65f, sR.y - 0.05f, sR.z + fz * 0.65f);
              tip = V3(hand.x + fx * 0.55f, hand.y, hand.z + fz * 0.55f);
          }
          else if (D.strikeT > 0.0f) {
              hand = V3(sR.x + fx * 0.55f, sR.y + 0.15f, sR.z + fz * 0.55f);
              tip = V3(hand.x + fx * 1.25f, hand.y - 0.45f, hand.z + fz * 1.25f);
          }
          else if (D.es == E_WINDUP_SWORD) {
              hand = V3(sR.x + fx * (0.15f - 0.1f * p), sR.y + 0.15f + 0.5f * p, sR.z + fz * (0.15f - 0.1f * p));
              tip = V3(hand.x - fx * 0.55f * p + fx * 0.1f * (1.0f - p), hand.y + 0.7f, hand.z - fz * 0.55f * p + fz * 0.1f * (1.0f - p));
              blade = (Color){ (unsigned char)(225 + 30 * p), (unsigned char)(228 - 150 * p), (unsigned char)(238 - 170 * p), 255 };
          }
          else if (D.es == E_WINDUP_SLAM) {
              hand = V3(sR.x, sR.y + 0.7f, sR.z);
              tip = V3(hand.x, hand.y + 0.9f, hand.z);
          }
          else if (D.estun > 0.0f) {
              hand = V3(sR.x + fx * 0.1f, sR.y - 0.5f, sR.z + fz * 0.1f);
              tip = V3(hand.x + fx * 0.6f, hand.y - 0.2f, hand.z + fz * 0.6f);
          }
          else {
              hand = V3(sR.x + fx * 0.3f, sR.y - 0.3f, sR.z + fz * 0.3f);
              tip = V3(hand.x + fx * 0.2f, hand.y + 0.85f, hand.z + fz * 0.2f);
          }
          DrawCylinderEx(sR, hand, 0.08f, 0.07f, 8, armor);
          if (gun) {
              DrawCylinderEx(hand, tip, 0.05f, 0.04f, 6, (Color) { 40, 40, 44, 255 });
              if (D.gunT > 0.4f) DrawSphere(V3(tip.x + fx * 0.1f, tip.y, tip.z + fz * 0.1f), 0.13f, (Color) { 255, 220, 90, 255 });
          }
          else {
              DrawCylinderEx(hand, tip, 0.04f, 0.015f, 6, blade);
              DrawCylinderEx(V3(hand.x - rx * 0.12f, hand.y, hand.z - rz * 0.12f), V3(hand.x + rx * 0.12f, hand.y, hand.z + rz * 0.12f), 0.025f, 0.025f, 6, trim);
              if (D.es == E_WINDUP_SWORD && p > 0.6f) DrawSphere(tip, 0.12f, (Color) { 255, 60, 40, 200 });
          }
      }
      
      static void DrawDuelWorld(void)
      {
          ClearBackground((Color) { 16, 18, 30, 255 });
          BeginMode3D(gCam);
      
          for (int r = 0; r < 8; r++)
              for (int c = 0; c < 8; c++) {
                  bool light = ((r + c) & 1) == 0;
                  DrawCube(V3(-10.5f + c * 3.0f, -0.1f, -10.5f + r * 3.0f), 3.0f, 0.2f, 3.0f,
                      light ? (Color) { 214, 192, 160, 255 } : (Color) { 150, 112, 82, 255 });
              }
          Color wall = (Color){ 66, 54, 52, 255 };
          DrawCube(V3(0.0f, 2.0f, -12.5f), 26.0f, 4.0f, 1.0f, wall);
          DrawCube(V3(0.0f, 2.0f, 12.5f), 26.0f, 4.0f, 1.0f, wall);
          DrawCube(V3(-12.5f, 2.0f, 0.0f), 1.0f, 4.0f, 26.0f, wall);
          DrawCube(V3(12.5f, 2.0f, 0.0f), 1.0f, 4.0f, 26.0f, wall);
          float flick = sinf((float)GetTime() * 9.0f) * 0.04f;
          for (int i = 0; i < 4; i++) {
              float z = -9.0f + i * 6.0f;
              DrawCylinder(V3(-11.8f, 1.2f, z), 0.08f, 0.08f, 1.0f, 6, (Color) { 90, 60, 40, 255 });
              DrawSphere(V3(-11.8f, 2.0f + flick, z), 0.22f, (Color) { 255, 150, 40, 255 });
              DrawCylinder(V3(11.8f, 1.2f, z), 0.08f, 0.08f, 1.0f, 6, (Color) { 90, 60, 40, 255 });
              DrawSphere(V3(11.8f, 2.0f + flick, z), 0.22f, (Color) { 255, 150, 40, 255 });
          }
          for (int i = 0; i < 2; i++) {
              DrawCylinder(V3(PILLARS[i][0], 0.0f, PILLARS[i][1]), PILLAR_R, PILLAR_R, 3.0f, 16, (Color) { 120, 115, 125, 255 });
              DrawCube(V3(PILLARS[i][0], 3.1f, PILLARS[i][1]), 2.2f, 0.25f, 2.2f, (Color) { 90, 86, 96, 255 });
              DrawCube(V3(PILLARS[i][0], 0.1f, PILLARS[i][1]), 2.2f, 0.2f, 2.2f, (Color) { 90, 86, 96, 255 });
          }
      
          DrawEnemy();
      
          for (int i = 0; i < MAX_BULLETS; i++) {
              Bullet* b = &D.bl[i];
              if (!b->active) continue;
              Color c = b->fromPlayer ? (Color) { 120, 220, 255, 255 } : (Color) { 255, 140, 40, 255 };
              DrawSphere(V3(b->x, b->y, b->z), 0.10f, c);
              DrawLine3D(V3(b->x, b->y, b->z), V3(b->x - b->vx * 0.04f, b->y - b->vy * 0.04f, b->z - b->vz * 0.04f), c);
          }
          if (D.trT > 0.0f) DrawLine3D(D.trA, D.trB, (Color) { 255, 235, 130, 255 });
      
          if (D.es == E_WINDUP_SLAM && D.estun <= 0.0f) {
              float pulse = 0.5f + 0.5f * sinf((float)GetTime() * 14.0f);
              for (int k = 1; k <= 3; k++)
                  DrawCircle3D(V3(D.slamX, 0.04f, D.slamZ), 3.4f * k / 3.0f, V3(1.0f, 0.0f, 0.0f), 90.0f,
                      (Color) {
                  255, 50, 40, (unsigned char)(120 + 120 * pulse)
              });
          }
          if (D.slamFx > 0.0f) {
              float rr = 3.4f + (0.45f - D.slamFx) * 6.0f;
              DrawCircle3D(V3(D.slamX, 0.06f, D.slamZ), rr, V3(1.0f, 0.0f, 0.0f), 90.0f, (Color) { 255, 120, 60, 255 });
              DrawCircle3D(V3(D.slamX, 0.06f, D.slamZ), rr * 0.9f, V3(1.0f, 0.0f, 0.0f), 90.0f, (Color) { 255, 200, 80, 255 });
          }
          if (D.shockFx > 0.0f) {
              float rr = 4.6f * (1.0f - D.shockFx / 0.45f);
              for (int k = 0; k < 3; k++)
                  DrawCircle3D(V3(D.px, 0.06f + k * 0.05f, D.pz), rr - k * 0.12f, V3(1.0f, 0.0f, 0.0f), 90.0f, (Color) { 120, 220, 255, 255 });
          }
          EndMode3D();
      }
      
      /* ----- 2D overlay : viewmodel + HUD ----- */
      static void DrawBladeScreen(Vector2 base, float angDeg, float len, unsigned char a)
      {
          float r = angDeg * PI_F / 180.0f;
          Vector2 tip = { base.x + sinf(r) * len, base.y - cosf(r) * len };
          DrawLineEx(base, tip, 28.0f, (Color) { 140, 150, 172, a });
          DrawLineEx(base, tip, 10.0f, (Color) { 238, 242, 252, a });
          Vector2 g = { base.x + sinf(r) * len * 0.22f, base.y - cosf(r) * len * 0.22f };
          Vector2 perp = { cosf(r), sinf(r) };
          DrawLineEx((Vector2) { g.x - perp.x * 55.0f, g.y - perp.y * 55.0f },
              (Vector2) {
              g.x + perp.x * 55.0f, g.y + perp.y * 55.0f
          }, 14.0f, (Color) { 200, 165, 60, a });
      }
      
      static void DrawSwordViewmodel(void)
      {
          float t = (float)GetTime();
          Vector2 base = { SCREEN_W * 0.80f, SCREEN_H + 30.0f };
          float ang = 16.0f + sinf(t * 2.0f) * 1.5f, len = 470.0f;
          if (D.guarding) { base.x = SCREEN_W * 0.60f; ang = -38.0f; len = 520.0f; }
          float prog = -1.0f;
          if (D.swingT > 0.0f) prog = 1.0f - D.swingT / 0.38f;
          if (prog >= 0.0f) {
              for (int k = 3; k >= 1; k--) {
                  float pe = ClampF(prog - 0.06f * k, 0.0f, 1.0f);
                  float e2 = pe * pe * (3.0f - 2.0f * pe);
                  DrawBladeScreen((Vector2) { base.x - 80.0f * e2, base.y }, 40.0f - 120.0f * e2, len, (unsigned char)(40 + 10 * (4 - k)));
              }
              float e = prog * prog * (3.0f - 2.0f * prog);
              base.x -= 80.0f * e;
              ang = 40.0f - 120.0f * e;
          }
          DrawBladeScreen(base, ang, len, 255);
          if (D.guarding && D.parryT > 0.0f)
              DrawCircleV((Vector2) { SCREEN_W * 0.5f, SCREEN_H * 0.45f }, 70.0f + D.parryT * 120.0f, (Color) { 120, 220, 255, 70 });
      }
      
      static void DrawGunViewmodel(void)
      {
          float rec = D.fireT > 0.0f ? 18.0f * (D.fireT / 0.1f) : 0.0f;
          float low = 0.0f;
          if (D.reloadT > 0.0f) low = 200.0f * sinf(PI_F * (1.0f - D.reloadT / 1.2f));
          float ox = SCREEN_W - 480.0f, oy = SCREEN_H - 200.0f + rec + low;
          DrawRectangle((int)ox + 100, (int)oy + 18, 130, 60, (Color) { 50, 52, 60, 255 });          /* slide */
          DrawRectangle((int)ox + 100, (int)oy + 18, 130, 12, (Color) { 90, 94, 108, 255 });
          DrawRectangle((int)ox - 20, (int)oy + 30, 130, 30, (Color) { 36, 38, 44, 255 });           /* barrel */
          DrawRectangle((int)ox + 180, (int)oy + 70, 56, 140, (Color) { 70, 48, 36, 255 });          /* grip */
          DrawRectangle((int)ox + 120, (int)oy + 74, 60, 8, (Color) { 30, 30, 34, 255 });
          if (D.fireT > 0.06f) {
              DrawCircleV((Vector2) { ox - 30.0f, oy + 44.0f }, 34.0f, (Color) { 255, 220, 90, 230 });
              DrawCircleV((Vector2) { ox - 30.0f, oy + 44.0f }, 18.0f, (Color) { 255, 255, 220, 255 });
          }
      }
      
      static void DrawBar(float x, float y, float w, float h, float frac, Color c)
      {
          DrawRectangle((int)x - 2, (int)y - 2, (int)w + 4, (int)h + 4, (Color) { 0, 0, 0, 190 });
          DrawRectangle((int)x, (int)y, (int)w, (int)h, (Color) { 50, 20, 20, 255 });
          DrawRectangle((int)x, (int)y, (int)(w * ClampF(frac, 0.0f, 1.0f)), (int)h, c);
      }
      
      static void DrawAbility(float x, float y, const char* key, const char* name, float cdLeft, float cdMax, Color c)
      {
          Rectangle r = { x, y, 74.0f, 74.0f };
          DrawRectangleRounded(r, 0.18f, 6, (Color) { 20, 22, 32, 225 });
          if (cdLeft > 0.0f && cdMax > 0.0f) {
              float f = ClampF(cdLeft / cdMax, 0.0f, 1.0f);
              DrawRectangle((int)x + 3, (int)(y + 3 + 68.0f * (1.0f - f)), 68, (int)(68.0f * f), (Color) { 0, 0, 0, 170 });
              DrawTextC(TextFormat("%.1f", cdLeft), (int)x + 37, (int)y + 24, 20, LIGHTGRAY);
          }
          else {
              DrawRectangleLinesEx(r, 3.0f, c);
          }
          DrawText(key, (int)x + 7, (int)y + 5, 20, c);
          DrawTextC(name, (int)x + 37, (int)y + 54, 14, LIGHTGRAY);
      }
      
      static void DrawDuelHUD(void)
      {
          /* damage vignette */
          if (D.hurtT > 0.0f)
              DrawRectangle(0, 0, SCREEN_W, SCREEN_H, (Color) { 255, 0, 0, (unsigned char)(100 * ClampF(D.hurtT / 0.35f, 0.0f, 1.0f)) });
          if (D.ultT > 0.0f)
              DrawRectangleLinesEx((Rectangle) { 0, 0, SCREEN_W, SCREEN_H }, 8.0f, (Color) { 255, 190, 60, 150 });
      
          /* sparks (hit / parry) */
          if (D.sparkT > 0.0f) {
              for (int i = 0; i < 12; i++) {
                  float a = i * PI_F / 6.0f, l = 30.0f + D.sparkT * 220.0f;
                  Vector2 c = { SCREEN_W * 0.5f, SCREEN_H * 0.45f };
                  DrawLineEx((Vector2) { c.x + cosf(a) * 20.0f, c.y + sinf(a) * 20.0f },
                      (Vector2) {
                      c.x + cosf(a) * l, c.y + sinf(a) * l
                  }, 4.0f, (Color) { 255, 230, 120, 255 });
              }
          }
      
          /* viewmodel */
          if (D.weapon == 0) DrawSwordViewmodel(); else DrawGunViewmodel();
      
          /* crosshair */
          float cx = SCREEN_W * 0.5f, cy = SCREEN_H * 0.5f;
          if (D.weapon == 1) {
              DrawLine((int)cx - 14, (int)cy, (int)cx - 5, (int)cy, RAYWHITE);
              DrawLine((int)cx + 5, (int)cy, (int)cx + 14, (int)cy, RAYWHITE);
              DrawLine((int)cx, (int)cy - 14, (int)cx, (int)cy - 5, RAYWHITE);
              DrawLine((int)cx, (int)cy + 5, (int)cx, (int)cy + 14, RAYWHITE);
          }
          DrawCircleV((Vector2) { cx, cy }, 2.5f, RAYWHITE);
      
          /* enemy telegraph marker */
          {
              Vector3 hp = V3(D.ex, D.ey + 2.6f, D.ez);
              Vector3 f = DuelForward3();
              float dot = (hp.x - gCam.position.x) * f.x + (hp.y - gCam.position.y) * f.y + (hp.z - gCam.position.z) * f.z;
              if (dot > 0.0f && D.phase == 1) {
                  Vector2 s = GetWorldToScreen(hp, gCam);
                  if (D.estun > 0.0f) DrawTextC("STUNNED", (int)s.x, (int)s.y - 10, 22, (Color) { 130, 180, 255, 255 });
                  else if (D.es == E_WINDUP_SWORD) DrawTextC("!", (int)s.x, (int)s.y - 30, 56, (Color) { 255, 60, 40, 255 });
                  else if (D.es == E_WINDUP_SHOT) DrawTextC("!", (int)s.x, (int)s.y - 30, 56, (Color) { 255, 170, 40, 255 });
                  else if (D.es == E_WINDUP_SLAM) DrawTextC("SLAM!", (int)s.x, (int)s.y - 20, 30, (Color) { 255, 80, 40, 255 });
              }
          }
      
          /* enemy bar */
          {
              const char* en = TextFormat("%s %s  (Lv %d)", D.ef.side > 0 ? "White" : "Black", PNAME[D.ef.piece], G.aiLevel);
              DrawPieceGlyph(D.ef.side * D.ef.piece, SCREEN_W / 2 - 220.0f, 36.0f, 18.0f, 1.0f);
              DrawText(en, SCREEN_W / 2 - 192, 14, 20, RAYWHITE);
              DrawBar(SCREEN_W / 2 - 192, 40, 384, 16, D.ehp / D.emax, (Color) { 210, 60, 60, 255 });
          }
          DrawText(TextFormat("TIME %02d", (int)ceilf(D.timeLeft)), SCREEN_W - 130, 16, 26, D.timeLeft < 15.0f ? (Color) { 255, 90, 80, 255 } : RAYWHITE);
          DrawText(D.playerIsAttacker ? "You: ATTACKER" : "You: DEFENDER", 20, 16, 20, LIGHTGRAY);
      
          /* player bar */
          DrawPieceGlyph(D.pf.side * D.pf.piece, 40.0f, SCREEN_H - 76.0f, 18.0f, 1.0f);
          DrawText(TextFormat("YOU - %s %s", D.pf.side > 0 ? "White" : "Black", PNAME[D.pf.piece]), 66, SCREEN_H - 92, 20, RAYWHITE);
          DrawBar(66, SCREEN_H - 66, 360, 18, D.php / D.pmax, (Color) { 90, 200, 90, 255 });
          DrawText(TextFormat("%d / %d", (int)ceilf(D.php), (int)D.pmax), 70, SCREEN_H - 40, 18, LIGHTGRAY);
      
          /* abilities */
          float ax = SCREEN_W / 2 - 135.0f, ay = SCREEN_H - 100.0f;
          DrawAbility(ax, ay, "Q", "Dash", D.dashCdT, D.pf.dashCd, (Color) { 120, 220, 255, 255 });
          DrawAbility(ax + 90.0f, ay, "E", "Shock", D.shockCdT, 8.0f, (Color) { 120, 220, 255, 255 });
          {
              Rectangle r = { ax + 180.0f, ay, 74.0f, 74.0f };
              DrawRectangleRounded(r, 0.18f, 6, (Color) { 20, 22, 32, 225 });
              float f = D.ultT > 0.0f ? D.ultT / 7.0f : D.ult / 100.0f;
              DrawRectangle((int)r.x + 3, (int)(r.y + 3 + 68.0f * (1.0f - f)), 68, (int)(68.0f * f), D.ultT > 0.0f ? (Color) { 255, 190, 60, 210 } : (Color) { 160, 110, 40, 200 });
              if (D.ult >= 100.0f && D.ultT <= 0.0f) DrawRectangleLinesEx(r, 3.0f, (Color) { 255, 200, 60, 255 });
              DrawText("F", (int)r.x + 7, (int)r.y + 5, 20, (Color) { 255, 200, 60, 255 });
              DrawTextC(D.ultT > 0.0f ? "ACTIVE" : "Overdrive", (int)r.x + 37, (int)r.y + 54, 14, LIGHTGRAY);
          }
      
          /* weapon info */
          if (D.weapon == 0) DrawText("[1] SWORD", SCREEN_W - 200, 60, 22, RAYWHITE);
          else DrawText(D.reloadT > 0.0f ? "[2] GUN  reloading..." : TextFormat("[2] GUN  %d / 12", D.ammo), SCREEN_W - 260, 60, 22, RAYWHITE);
          DrawText(D.guarding ? "GUARD" : "", SCREEN_W - 200, 90, 22, (Color) { 120, 220, 255, 255 });
      
          /* message */
          if (D.msgT > 0.0f)
              DrawTextC(D.msg, SCREEN_W / 2, 150, 46, (Color) { 255, 220, 90, (unsigned char)(255 * ClampF(D.msgT / 0.5f, 0.0f, 1.0f)) });
      }
      
      static void DuelFrame(float dt)
      {
          ClearBackground((Color) { 16, 18, 30, 255 });
      
          if (IsKeyPressed(KEY_ESCAPE) && D.phase != 2) {
              D.paused = !D.paused;
              if (D.paused) EnableCursor(); else { DisableCursor(); D.skipLook = 3; }
          }
          if (!D.paused) {
              D.phaseT += dt;
              if (UpdateDuel(dt)) return;
          }
      
          UpdateCam();
          DrawDuelWorld();
          DrawDuelHUD();
      
          /* intro overlay */
          if (D.phase == 0) {
              float a = ClampF((2.8f - D.phaseT) / 0.4f, 0.0f, 1.0f);
              DrawRectangle(0, 0, SCREEN_W, SCREEN_H, (Color) { 0, 0, 0, (unsigned char)(150 * a) });
              DrawTextC("DUEL!", SCREEN_W / 2, 90, 110, (Color) { 255, 90, 70, (unsigned char)(255 * a) });
              int att = G.hist[G.nply].b[D.move.from];
              int defp = G.hist[G.nply].b[D.move.to];
              if (defp == 0) defp = G.hist[G.nply].b[RowOf(D.move.from) * 8 + ColOf(D.move.to)];
              DrawPieceGlyph(att, SCREEN_W / 2 - 220.0f, 290.0f, 50.0f, a);
              DrawPieceGlyph(defp, SCREEN_W / 2 + 220.0f, 290.0f, 50.0f, a);
              DrawTextC("VS", SCREEN_W / 2, 262, 56, Fade(RAYWHITE, a));
              DrawTextC(TextFormat("%s attacks %s", PNAME[abs(att)], PNAME[abs(defp)]), SCREEN_W / 2, 360, 26, Fade(LIGHTGRAY, a));
              int cnt = (int)ceilf(2.8f - D.phaseT);
              if (cnt > 3) cnt = 3;
              if (cnt >= 1) DrawTextC(TextFormat("%d", cnt), SCREEN_W / 2, 420, 110, Fade(GOLD, a));
              DrawTextC("WASD move  |  Mouse look  |  LMB attack  |  RMB guard (tap right before a hit = PARRY)",
                  SCREEN_W / 2, 600, 22, Fade(RAYWHITE, a));
              DrawTextC("Q dash  |  E shockwave  |  F overdrive  |  1/2/Tab/Wheel weapon  |  R reload  |  ESC pause",
                  SCREEN_W / 2, 632, 22, Fade(RAYWHITE, a));
          }
          else if (D.phase == 1 && D.phaseT < 0.8f) {
              DrawTextC("FIGHT!", SCREEN_W / 2, 200, 90, (Color) { 255, 220, 90, (unsigned char)(255 * (1.0f - D.phaseT / 0.8f)) });
          }
      
          /* result overlay */
          if (D.phase == 2) {
              float a = ClampF(D.phaseT / 0.6f, 0.0f, 1.0f);
              DrawRectangle(0, 0, SCREEN_W, SCREEN_H, (Color) { 0, 0, 0, (unsigned char)(150 * a) });
              DrawTextC(D.playerWon ? "VICTORY" : "DEFEATED", SCREEN_W / 2, 200, 120,
                  D.playerWon ? (Color) { 255, 220, 90, (unsigned char)(255 * a) } : (Color) { 230, 70, 60, (unsigned char)(255 * a) });
              DrawTextC(D.attackerWins ? "The capture succeeds!" : "The attacker is slain - the defender holds the square!",
                  SCREEN_W / 2, 360, 30, Fade(RAYWHITE, a));
              if (D.phaseT > 1.3f)
                  DrawTextC("Click or press ENTER to return to the board", SCREEN_W / 2, 470, 24,
                      Fade(LIGHTGRAY, 0.5f + 0.5f * sinf((float)GetTime() * 4.0f)));
          }
      
          /* pause */
          if (D.paused) {
              Rectangle b = { SCREEN_W / 2 - 190.0f, SCREEN_H / 2 - 120.0f, 380.0f, 240.0f };
              DrawDialog(b, "Duel paused");
              if (UiButton((Rectangle) { b.x + 40, b.y + 80, 300, 48 }, "Resume", true, false)) { D.paused = false; DisableCursor(); D.skipLook = 3; }
              if (UiButton((Rectangle) { b.x + 40, b.y + 140, 300, 48 }, "Forfeit duel (lose)", true, false)) {
                  D.paused = false;
                  D.php = 0.0f;
                  EndDuel(false);
              }
          }
      }
      
      /* ---------------------------------------------------------------------
       *  GAME
       * ------------------------------------------------------------------- */
      static void DrawPlayerCard(const Pos* cur, int side, Rectangle r, bool over)
      {
          int ai = AISide();
          DrawRectangleRounded(r, 0.12f, 8, (Color) { 30, 32, 44, 255 });
          if (cur->turn == side && !over)
              DrawRectangleRounded((Rectangle) { r.x, r.y, 6, r.height }, 0.5f, 4, (Color) { 120, 200, 100, 255 });
      
          const char* name;
          if (ai == side) name = TextFormat("AI  (Lv %d)", G.aiLevel);
          else if (ai != 0) name = "You";
          else name = side == SIDE_W ? "White" : "Black";
      
          DrawPieceGlyph(side * KING, r.x + 30, r.y + 28, 14, 1.0f);
          DrawText(name, (int)r.x + 54, (int)r.y + 18, 22, RAYWHITE);
      
          const char* ck = G.tc > 0 ? FmtClock(G.clock[side == SIDE_W ? 0 : 1]) : "--:--";
          bool low = G.tc > 0 && G.clock[side == SIDE_W ? 0 : 1] < 30.0f;
          DrawText(ck, (int)(r.x + r.width - MeasureText(ck, 30) - 14), (int)r.y + 12, 30,
              low ? (Color) { 240, 90, 80, 255 } : RAYWHITE);
      
          int miss[7];
          CountMissing(cur, -side, miss);
          float x = r.x + 16, y = r.y + r.height - 22;
          for (int t = QUEEN; t >= PAWN; t--)
              for (int k = 0; k < miss[t]; k++) { DrawPieceGlyph(-side * t, x + 9, y, 9, 1.0f); x += 15; }
          int diff = MaterialSum(cur, side) - MaterialSum(cur, -side);
          if (diff > 0) DrawText(TextFormat("+%d", diff / 100), (int)(r.x + r.width - 40), (int)(r.y + r.height - 30), 20, GOLD);
      }
      
      static void GameFrame(float dt)
      {
          int ai = AISide();
      
          /* ------------- timers ------------- */
          if (G.toastT > 0) G.toastT -= dt;
          if (G.animOn) { G.animT += dt / 0.16f; if (G.animT >= 1.0f) G.animOn = false; }
      
          const Pos* cur = &G.hist[G.nply];
          bool over = G.result != RES_NONE;
      
          if (!over && G.modal != MODAL_PAUSE && G.tc > 0) {
              int idx = cur->turn == SIDE_W ? 0 : 1;
              G.clock[idx] -= dt;
              if (G.clock[idx] <= 0.0f) {
                  G.clock[idx] = 0.0f;
                  SetResult(cur->turn == SIDE_W ? RES_BLACK : RES_WHITE, "Time out");
              }
          }
          over = G.result != RES_NONE;
      
          /* ------------- AI ------------- */
          if (!over && G.modal == MODAL_NONE && ai != 0 && cur->turn == ai && !G.animOn) {
              G.aiTimer += dt;
              if (G.aiTimer > 0.45f) {
                  G.aiTimer = 0.0f;
                  Move m = ChooseAIMove(cur, G.aiLevel);
                  TryMove(m);
              }
          }
          cur = &G.hist[G.nply];
          over = G.result != RES_NONE;
      
          /* ------------- keyboard ------------- */
          bool blocked = (G.modal != MODAL_NONE) || (over && !G.hideOverlay);
          if (IsKeyPressed(KEY_ESCAPE)) {
              if (G.modal == MODAL_NONE) { if (!over) G.modal = MODAL_PAUSE; else G.hideOverlay = false; }
              else if (G.modal != MODAL_NONE) G.modal = MODAL_NONE;
              blocked = true;
          }
          else if (!blocked) {
              if (IsKeyPressed(KEY_F)) G.flipped = !G.flipped;
              if (IsKeyPressed(KEY_H)) G.hints = !G.hints;
              if (!over) {
                  if (IsKeyPressed(KEY_U)) DoUndo();
                  if (IsKeyPressed(KEY_R)) G.modal = MODAL_RESIGN;
                  if (IsKeyPressed(KEY_D)) AskDraw();
              }
          }
          cur = &G.hist[G.nply];
          over = G.result != RES_NONE;
          blocked = (G.modal != MODAL_NONE) || (over && !G.hideOverlay);
      
          /* ------------- board click ------------- */
          if (!blocked && !over && IsMouseButtonPressed(MOUSE_BUTTON_LEFT) && !gClickUsed) {
              int sq = XYToSq(GetMousePosition());
              bool humanTurn = (ai == 0 || cur->turn != ai);
              if (sq >= 0 && humanTurn) {
                  gClickUsed = true;
                  if (Sign(cur->b[sq]) == cur->turn) {
                      G.selected = (G.selected == sq) ? -1 : sq;
                  }
                  else if (G.selected >= 0) {
                      int cnt = 0;
                      Move found = { 0, 0, 0 };
                      for (int i = 0; i < G.nlegal; i++)
                          if (G.legal[i].from == G.selected && G.legal[i].to == sq) { found = G.legal[i]; cnt++; }
                      if (cnt > 0) {
                          if (found.promo) { G.promoBase = found; G.modal = MODAL_PROMO; }
                          else TryMove(found);
                      }
                      else G.selected = -1;
                  }
              }
          }
          cur = &G.hist[G.nply];
          over = G.result != RES_NONE;
          blocked = (G.modal != MODAL_NONE) || (over && !G.hideOverlay);
      
          /* ================= DRAW ================= */
          ClearBackground((Color) { 28, 30, 40, 255 });
          DrawText("CHESS", BOARD_X, 14, 30, RAYWHITE);
          const char* info = ai ? TextFormat("vs AI (Lv %d)  -  %s  -  Duels %s", G.aiLevel, TC_NAME[G.tc], G.duels ? "ON" : "OFF")
              : TextFormat("2 Players  -  %s  -  Duels %s", TC_NAME[G.tc], G.duels ? "ON" : "OFF");
          DrawText(info, BOARD_X + TILE * 8 + 14 - MeasureText(info, 20), 20, 20, GRAY);
      
          /* board frame + squares */
          DrawRectangle(BOARD_X - 14, BOARD_Y - 14, TILE * 8 + 28, TILE * 8 + 28, (Color) { 58, 42, 34, 255 });
          for (int sq = 0; sq < 64; sq++) {
              Vector2 p = SqToXY(sq);
              bool light = ((RowOf(sq) + ColOf(sq)) & 1) == 0;
              DrawRectangle((int)p.x, (int)p.y, TILE, TILE, light ? (Color) { 240, 217, 181, 255 } : (Color) { 181, 136, 99, 255 });
          }
          /* last move + selection + check */
          if (G.nply > 0) {
              Move lm = G.mv[G.nply - 1];
              Vector2 a = SqToXY(lm.from), b = SqToXY(lm.to);
              DrawRectangle((int)a.x, (int)a.y, TILE, TILE, (Color) { 205, 210, 106, 140 });
              DrawRectangle((int)b.x, (int)b.y, TILE, TILE, (Color) { 205, 210, 106, 170 });
          }
          if (G.selected >= 0) {
              Vector2 p = SqToXY(G.selected);
              DrawRectangle((int)p.x, (int)p.y, TILE, TILE, (Color) { 246, 246, 105, 170 });
          }
          if (InCheck(cur)) {
              int ks = FindKing(cur, cur->turn);
              if (ks >= 0) {
                  Vector2 p = SqToXY(ks);
                  DrawGlow(p.x + TILE / 2.0f, p.y + TILE / 2.0f, TILE * 0.62f, (Color) { 230, 40, 40, 255 });
              }
          }
          /* coordinates */
          for (int i = 0; i < 8; i++) {
              int fileIdx = G.flipped ? 7 - i : i;
              int rankIdx = G.flipped ? 7 - i : i;
              bool lightB = (((7 - i) + i) & 1) == 0;  /* placeholder, recomputed below */
              (void)lightB;
              /* file letter at the bottom row, rank number at the left column */
              int bottomR = G.flipped ? 0 : 7;
              int leftC = G.flipped ? 7 : 0;
              bool lightFile = ((bottomR + fileIdx) & 1) == 0;
              bool lightRank = ((rankIdx + leftC) & 1) == 0;
              char f[2] = { (char)('a' + fileIdx), 0 };
              char rk[2] = { (char)('8' - rankIdx), 0 };
              DrawText(f, BOARD_X + i * TILE + TILE - 14, BOARD_Y + 7 * TILE + TILE - 18, 16,
                  lightFile ? (Color) { 181, 136, 99, 255 } : (Color) { 240, 217, 181, 255 });
              DrawText(rk, BOARD_X + 5, BOARD_Y + i * TILE + 4, 16,
                  lightRank ? (Color) { 181, 136, 99, 255 } : (Color) { 240, 217, 181, 255 });
          }
          /* move hints */
          if (G.selected >= 0 && G.hints && !over) {
              for (int i = 0; i < G.nlegal; i++) {
                  if (G.legal[i].from != G.selected || (G.legal[i].promo && G.legal[i].promo != QUEEN)) continue;
                  int to = G.legal[i].to;
                  Vector2 p = SqToXY(to);
                  Vector2 c = { p.x + TILE / 2.0f, p.y + TILE / 2.0f };
                  bool cap = cur->b[to] != 0 || (abs(cur->b[G.selected]) == PAWN && ColOf(to) != ColOf(G.selected));
                  if (cap) DrawRing(c, TILE * 0.40f, TILE * 0.47f, 0, 360, 32,
                      G.duels ? (Color) { 210, 40, 40, 190 } : (Color) { 0, 0, 0, 70 });
                  else DrawCircleV(c, TILE * 0.16f, (Color) { 0, 0, 0, 60 });
              }
          }
          /* pieces */
          for (int sq = 0; sq < 64; sq++) {
              if (!cur->b[sq]) continue;
              if (G.animOn && sq == G.animTo) continue;
              Vector2 p = SqToXY(sq);
              DrawPieceGlyph(cur->b[sq], p.x + TILE / 2.0f, p.y + TILE / 2.0f, TILE * 0.38f, 1.0f);
          }
          if (G.animOn) {
              float k = G.animT;
              k = k * k * (3.0f - 2.0f * k);
              Vector2 a = SqToXY(G.animFrom), b = SqToXY(G.animTo);
              DrawPieceGlyph(G.animPiece, a.x + (b.x - a.x) * k + TILE / 2.0f,
                  a.y + (b.y - a.y) * k + TILE / 2.0f, TILE * 0.38f, 1.0f);
          }
      
          /* ------------- side panel ------------- */
          int topSide = G.flipped ? SIDE_W : SIDE_B;
          DrawPlayerCard(cur, topSide, (Rectangle) { PANEL_X, 60, PANEL_W, 80 }, over);
          DrawPlayerCard(cur, -topSide, (Rectangle) { PANEL_X, 620, PANEL_W, 80 }, over);
      
          /* move list */
          Rectangle lr = { PANEL_X, 150, PANEL_W, 240 };
          DrawRectangleRounded(lr, 0.04f, 8, (Color) { 30, 32, 44, 255 });
          DrawText("Moves", PANEL_X + 12, 157, 18, GRAY);
          int totalRows = (G.nply + 1) / 2, visible = 8;
          int maxScroll = totalRows > visible ? totalRows - visible : 0;
          if (CheckCollisionPointRec(GetMousePosition(), lr)) {
              G.moveScroll += (int)GetMouseWheelMove();
          }
          if (G.moveScroll < 0) G.moveScroll = 0;
          if (G.moveScroll > maxScroll) G.moveScroll = maxScroll;
          int start = totalRows - visible - G.moveScroll;
          if (start < 0) start = 0;
          for (int i = start; i < start + visible && i < totalRows; i++) {
              int y = 182 + (i - start) * 24;
              int last = G.nply - 1;
              if (last == 2 * i)     DrawRectangle(PANEL_X + 54, y - 2, 120, 22, (Color) { 70, 80, 60, 255 });
              if (last == 2 * i + 1) DrawRectangle(PANEL_X + 184, y - 2, 120, 22, (Color) { 70, 80, 60, 255 });
              DrawText(TextFormat("%d.", i + 1), PANEL_X + 12, y, 18, GRAY);
              DrawText(G.san[2 * i], PANEL_X + 60, y, 18, RAYWHITE);
              if (2 * i + 1 < G.nply) DrawText(G.san[2 * i + 1], PANEL_X + 190, y, 18, RAYWHITE);
          }
      
          /* status */
          const char* status;
          Color stc = RAYWHITE;
          if (over) { status = ResultTitle(); stc = GOLD; }
          else if (ai != 0 && cur->turn == ai) status = "AI is thinking...";
          else if (InCheck(cur)) { status = cur->turn == SIDE_W ? "White is in CHECK!" : "Black is in CHECK!"; stc = (Color){ 240, 100, 90, 255 }; }
          else status = cur->turn == SIDE_W ? "White to move" : "Black to move";
          DrawTextC(status, PANEL_X + PANEL_W / 2, 397, 24, stc);
      
          /* buttons */
          bool inputOK = !blocked;
          if (!over) {
              if (UiButton((Rectangle) { PANEL_X, 432, 165, 40 }, "Undo (U)", inputOK&& G.nply > 0, false)) DoUndo();
              if (UiButton((Rectangle) { PANEL_X + 175, 432, 165, 40 }, "Flip (F)", inputOK, false)) G.flipped = !G.flipped;
              if (UiButton((Rectangle) { PANEL_X, 480, 165, 40 }, "Resign (R)", inputOK, false)) G.modal = MODAL_RESIGN;
              if (UiButton((Rectangle) { PANEL_X + 175, 480, 165, 40 }, "Draw (D)", inputOK, false)) AskDraw();
              if (UiButton((Rectangle) { PANEL_X, 528, 165, 40 }, "Hints (H)", inputOK, G.hints)) G.hints = !G.hints;
              if (UiButton((Rectangle) { PANEL_X + 175, 528, 165, 40 }, "Pause (ESC)", inputOK, false)) G.modal = MODAL_PAUSE;
          }
          else {
              if (UiButton((Rectangle) { PANEL_X, 432, 165, 40 }, "Rematch", inputOK, false)) NewGame();
              if (UiButton((Rectangle) { PANEL_X + 175, 432, 165, 40 }, "Main menu", inputOK, false)) GoScene(SCENE_MENU);
              if (UiButton((Rectangle) { PANEL_X, 480, 165, 40 }, "Flip (F)", inputOK, false)) G.flipped = !G.flipped;
              if (UiButton((Rectangle) { PANEL_X + 175, 480, 165, 40 }, "Result (ESC)", true, false)) G.hideOverlay = false;
          }
      
          /* toast */
          if (G.toastT > 0) {
              int w = MeasureText(G.toast, 20) + 30;
              DrawRectangleRounded((Rectangle) { BOARD_X + TILE * 4 - w / 2.0f, 718, (float)w, 32 }, 0.4f, 8, (Color) { 0, 0, 0, 200 });
              DrawTextC(G.toast, BOARD_X + TILE * 4, 724, 20, GOLD);
          }
      
          /* ------------- modals ------------- */
          float cx = BOARD_X + TILE * 4.0f, cy = BOARD_Y + TILE * 4.0f;
      
          if (G.modal == MODAL_PAUSE) {
              Rectangle b = { cx - 190, cy - 150, 380, 300 };
              DrawDialog(b, "Paused");
              if (UiButton((Rectangle) { b.x + 40, b.y + 70, 300, 44 }, "Resume", true, false)) G.modal = MODAL_NONE;
              if (UiButton((Rectangle) { b.x + 40, b.y + 124, 300, 44 }, "Restart game", true, false)) NewGame();
              if (UiButton((Rectangle) { b.x + 40, b.y + 178, 300, 44 }, "Main menu", true, false)) GoScene(SCENE_MENU);
              if (UiButton((Rectangle) { b.x + 40, b.y + 232, 300, 44 }, "Quit", true, false)) GoScene(SCENE_OUTRO);
          }
          else if (G.modal == MODAL_RESIGN) {
              Rectangle b = { cx - 190, cy - 90, 380, 180 };
              DrawDialog(b, "Resign this game?");
              if (UiButton((Rectangle) { b.x + 30, b.y + 100, 150, 46 }, "Yes", true, false)) DoResign();
              if (UiButton((Rectangle) { b.x + 200, b.y + 100, 150, 46 }, "No", true, false)) G.modal = MODAL_NONE;
          }
          else if (G.modal == MODAL_DRAW) {
              Rectangle b = { cx - 190, cy - 90, 380, 180 };
              DrawDialog(b, "Agree to a draw?");
              if (UiButton((Rectangle) { b.x + 30, b.y + 100, 150, 46 }, "Yes", true, false)) SetResult(RES_DRAW, "Draw by agreement");
              if (UiButton((Rectangle) { b.x + 200, b.y + 100, 150, 46 }, "No", true, false)) G.modal = MODAL_NONE;
          }
          else if (G.modal == MODAL_PROMO) {
              Rectangle b = { cx - 220, cy - 85, 440, 170 };
              DrawDialog(b, "Promote to:");
              static const int opts[4] = { QUEEN, ROOK, BISHOP, KNIGHT };
              for (int i = 0; i < 4; i++) {
                  Rectangle r = { b.x + 20 + i * 100, b.y + 70, 90, 90 };
                  bool hover = CheckCollisionPointRec(GetMousePosition(), r);
                  DrawRectangleRounded(r, 0.15f, 8, hover ? (Color) { 100, 106, 136, 255 } : (Color) { 62, 68, 92, 255 });
                  DrawPieceGlyph(cur->turn * opts[i], r.x + 45, r.y + 45, 32, 1.0f);
                  if (hover && IsMouseButtonPressed(MOUSE_BUTTON_LEFT) && !gClickUsed) {
                      gClickUsed = true;
                      Move m = G.promoBase;
                      m.promo = opts[i];
                      G.modal = MODAL_NONE;
                      TryMove(m);
                  }
              }
          }
      
          /* ------------- game-over overlay (outro of a game) ------------- */
          if (G.result != RES_NONE && !G.hideOverlay && G.modal == MODAL_NONE) {
              Rectangle b = { cx - 210, cy - 140, 420, 280 };
              DrawDialog(b, "GAME OVER");
              DrawTextC(ResultTitle(), (int)cx, (int)b.y + 64, 44, GOLD);
              DrawTextC(G.reason, (int)cx, (int)b.y + 118, 24, LIGHTGRAY);
              int secs = (int)(GetTime() - G.startTime);
              DrawTextC(TextFormat("Moves: %d     Time: %02d:%02d", (G.nply + 1) / 2, secs / 60, secs % 60),
                  (int)cx, (int)b.y + 152, 20, GRAY);
              if (UiButton((Rectangle) { b.x + 20, b.y + 190, 185, 38 }, "Rematch", true, false)) NewGame();
              if (UiButton((Rectangle) { b.x + 215, b.y + 190, 185, 38 }, "Main menu", true, false)) GoScene(SCENE_MENU);
              if (UiButton((Rectangle) { b.x + 20, b.y + 235, 185, 38 }, "View board", true, false)) G.hideOverlay = true;
              if (UiButton((Rectangle) { b.x + 215, b.y + 235, 185, 38 }, "Quit", true, false)) GoScene(SCENE_OUTRO);
          }
      }
      
      /* ---------------------------------------------------------------------
       *  OUTRO
       * ------------------------------------------------------------------- */
      static void OutroFrame(double t)
      {
          DrawBackdrop(t + 5.0);
          float a = ClampF((float)(t / 0.8), 0.0f, 1.0f);
          DrawTextC("Thanks for playing!", SCREEN_W / 2, 190, 72, Fade(RAYWHITE, a));
      
          float b = ClampF((float)((t - 0.6) / 0.8), 0.0f, 1.0f);
          DrawTextC(TextFormat("Games played: %d", gGames), SCREEN_W / 2, 320, 30, Fade(GOLD, b));
          DrawTextC(TextFormat("White wins: %d     Black wins: %d     Draws: %d", gWhiteWins, gBlackWins, gDraws),
              SCREEN_W / 2, 370, 26, Fade(LIGHTGRAY, b));
          DrawTextC(TextFormat("Total moves played: %d", (gTotalPlies + 1) / 2), SCREEN_W / 2, 415, 26, Fade(LIGHTGRAY, b));
      
          float c = ClampF((float)((t - 1.2) / 0.8), 0.0f, 1.0f);
          static const int row[6] = { ROOK, KNIGHT, BISHOP, QUEEN, KING, PAWN };
          for (int i = 0; i < 6; i++)
              DrawPieceGlyph((i % 2 == 0) ? row[i] : -row[i], SCREEN_W / 2 + (i - 2.5f) * 100.0f, 535, 34, c);
          DrawTextC("Made with C + raylib", SCREEN_W / 2, 610, 22, Fade(GRAY, c));
      
          if (t > 3.6) DrawRectangle(0, 0, SCREEN_W, SCREEN_H, Fade(BLACK, ClampF((float)((t - 3.6) / 0.6), 0.0f, 1.0f)));
          if (t > 4.2 || (t > 1.0 && (IsKeyPressed(KEY_ENTER) || IsKeyPressed(KEY_SPACE) || IsKeyPressed(KEY_ESCAPE) ||
              IsMouseButtonPressed(MOUSE_BUTTON_LEFT))))
              gQuit = true;
      }
      
      /* ---------------------------------------------------------------------
       *  main
       * ------------------------------------------------------------------- */
      int main(void)
      {
          SetConfigFlags(FLAG_MSAA_4X_HINT | FLAG_VSYNC_HINT);
          InitWindow(SCREEN_W, SCREEN_H, "Chess - raylib");
          SetExitKey(KEY_NULL);            /* ESC is used for the pause menu */
          SetTargetFPS(60);
      
          G.mode = 0;
          G.aiLevel = 2;
          G.tc = 0;
          G.hints = true;
          G.duels = true;
          NewGame();
          GoScene(SCENE_INTRO);
      
          while (!gQuit && !WindowShouldClose()) {
              float dt = GetFrameTime();
              if (dt > 0.1f) dt = 0.1f;
              double t = GetTime() - gSceneStart;
              gClickUsed = false;
      
              BeginDrawing();
              switch (gScene) {
              case SCENE_INTRO: IntroFrame(t); break;
              case SCENE_MENU:  MenuFrame(t);  break;
              case SCENE_GAME:  GameFrame(dt); break;
              case SCENE_DUEL:  DuelFrame(dt); break;
              case SCENE_OUTRO: OutroFrame(t); break;
              }
              EndDrawing();
          }
          CloseWindow();
          return 0;
      }
