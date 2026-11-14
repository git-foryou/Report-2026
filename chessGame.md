## 피지컬 체스 게임 코드

                  /* ======================================================================
                   *  CHESS  -  C + raylib
                   *
                   *  Scenes : INTRO -> MENU -> GAME -> (GAME OVER) -> MENU / OUTRO
                   *  Rules  : full legal-move generation, check / checkmate / stalemate,
                   *           castling, en passant, promotion, 50-move rule,
                   *           threefold repetition, insufficient material, time out
                   *  Extras : always vs. AI (play White or Black), AI = negamax + alpha-beta (3 levels),
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
                      if (G.mode != 1 && G.mode != 2) G.mode = 1;     /* the opponent is always the AI */
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
                  
                  static void GTri(Vector2 a, Vector2 b, Vector2 c, Color col) { DrawTriangle(a, b, c, col); DrawTriangle(a, c, b, col); }
                  static Vector2 GP(float cx, float cy, float s, float u, float v) { Vector2 p; p.x = cx + u * s; p.y = cy + v * s; return p; }
                  static void GBox(float cx, float cy, float s, float u0, float v0, float u1, float v1, Color col)
                  {
                      Vector2 sz; sz.x = (u1 - u0) * s; sz.y = (v1 - v0) * s;
                      DrawRectangleV(GP(cx, cy, s, u0, v0), sz, col);
                  }
                  static void GDisc(float cx, float cy, float s, float u, float v, float r, Color col) { DrawCircleV(GP(cx, cy, s, u, v), r * s, col); }
                  
                  /* piece silhouettes in unit coordinates (y grows downward); g = outline growth */
                  static void PieceShapes(int t, float cx, float cy, float s, Color col, float g)
                  {
                      GBox(cx, cy, s, -0.70f - g, 0.52f - g, 0.70f + g, 0.80f + g, col);
                      GBox(cx, cy, s, -0.52f - g, 0.40f - g, 0.52f + g, 0.56f + g, col);
                      switch (t) {
                      case PAWN:
                          GDisc(cx, cy, s, 0.0f, -0.48f, 0.27f + g, col);
                          GBox(cx, cy, s, -0.14f - g, -0.30f, 0.14f + g, 0.02f, col);
                          GTri(GP(cx, cy, s, 0.0f, -0.12f - g), GP(cx, cy, s, -0.40f - g, 0.44f + g), GP(cx, cy, s, 0.40f + g, 0.44f + g), col);
                          break;
                      case ROOK:
                          GBox(cx, cy, s, -0.30f - g, -0.30f - g, 0.30f + g, 0.44f, col);
                          GBox(cx, cy, s, -0.44f - g, -0.62f - g, 0.44f + g, -0.28f + g, col);
                          GBox(cx, cy, s, -0.44f - g, -0.82f - g, -0.26f + g, -0.60f, col);
                          GBox(cx, cy, s, -0.09f - g, -0.82f - g, 0.09f + g, -0.60f, col);
                          GBox(cx, cy, s, 0.26f - g, -0.82f - g, 0.44f + g, -0.60f, col);
                          break;
                      case BISHOP: {
                          Vector2 e = GP(cx, cy, s, 0.0f, -0.22f);
                          DrawEllipse((int)e.x, (int)e.y, (0.30f + g) * s, (0.46f + g) * s, col);
                          GDisc(cx, cy, s, 0.0f, -0.74f, 0.11f + g, col);
                          GTri(GP(cx, cy, s, 0.0f, 0.05f), GP(cx, cy, s, -0.40f - g, 0.44f + g), GP(cx, cy, s, 0.40f + g, 0.44f + g), col);
                      } break;
                      case QUEEN:
                          GTri(GP(cx, cy, s, 0.0f, -0.45f - g), GP(cx, cy, s, -0.42f - g, 0.44f + g), GP(cx, cy, s, 0.42f + g, 0.44f + g), col);
                          GBox(cx, cy, s, -0.44f - g, -0.58f - g, 0.44f + g, -0.38f + g, col);
                          GDisc(cx, cy, s, -0.40f, -0.66f, 0.10f + g, col);
                          GDisc(cx, cy, s, -0.20f, -0.76f, 0.10f + g, col);
                          GDisc(cx, cy, s, 0.0f, -0.82f, 0.10f + g, col);
                          GDisc(cx, cy, s, 0.20f, -0.76f, 0.10f + g, col);
                          GDisc(cx, cy, s, 0.40f, -0.66f, 0.10f + g, col);
                          break;
                      case KING:
                          GTri(GP(cx, cy, s, 0.0f, -0.35f - g), GP(cx, cy, s, -0.42f - g, 0.44f + g), GP(cx, cy, s, 0.42f + g, 0.44f + g), col);
                          GBox(cx, cy, s, -0.40f - g, -0.52f - g, 0.40f + g, -0.30f + g, col);
                          GBox(cx, cy, s, -0.07f - g, -0.98f - g, 0.07f + g, -0.50f, col);
                          GBox(cx, cy, s, -0.22f - g, -0.88f - g, 0.22f + g, -0.74f + g, col);
                          break;
                      case KNIGHT: {
                          static const float kx[9] = { -0.36f, 0.38f, 0.36f, 0.18f, 0.02f, -0.14f, -0.52f, -0.50f, -0.22f };
                          static const float ky[9] = { 0.44f, 0.44f, 0.00f, -0.40f, -0.72f, -0.56f, -0.12f, 0.06f, 0.02f };
                          float gs = 1.0f + g * 2.2f;
                          Vector2 c0 = GP(cx, cy, s, 0.0f, 0.05f);
                          for (int i = 0; i < 9; i++) {
                              int j = (i + 1) % 9;
                              GTri(c0, GP(cx, cy, s, kx[i] * gs, 0.05f + (ky[i] - 0.05f) * gs), GP(cx, cy, s, kx[j] * gs, 0.05f + (ky[j] - 0.05f) * gs), col);
                          }
                      } break;
                      default: break;
                      }
                  }
                  
                  static void DrawPieceGlyph(int piece, float cx, float cy, float rad, float alpha)
                  {
                      int t = abs(piece);
                      if (t < 1 || t > 6) return;
                      bool white = piece > 0;
                      float al = ClampF(alpha, 0.0f, 1.0f);
                      unsigned char a = (unsigned char)(255 * al);
                      Color edge = white ? (Color) { 70, 58, 48, a } : (Color) { 230, 220, 204, a };
                      Color fill = white ? (Color) { 250, 246, 236, a } : (Color) { 48, 48, 60, a };
                      Color det = white ? (Color) { 190, 150, 70, a } : (Color) { 200, 70, 70, a };
                      float s = rad * 0.98f;
                      DrawEllipse((int)cx + 2, (int)(cy + s * 0.86f), s * 0.85f, s * 0.2f, (Color) { 0, 0, 0, (unsigned char)(70 * al) });
                      PieceShapes(t, cx, cy, s, edge, 0.07f);
                      PieceShapes(t, cx, cy, s, fill, 0.0f);
                      GBox(cx, cy, s, -0.70f, 0.52f, 0.70f, 0.60f, det);                 /* accent stripe on the base */
                      if (t == KNIGHT) GDisc(cx, cy, s, -0.22f, -0.38f, 0.05f, edge);   /* eye */
                      if (t == BISHOP) GBox(cx, cy, s, -0.03f, -0.52f, 0.03f, -0.30f, det);
                      if (t == KING || t == QUEEN) GBox(cx, cy, s, -0.40f, -0.50f, 0.40f, -0.44f, det);
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
                  
                      DrawTextC("You play chess against the AI as", SCREEN_W / 2, 100, 22, LIGHTGRAY);
                      if (UiButton((Rectangle) { SCREEN_W / 2 - 300.0f, 130, 280, 56 }, "White (you move first)", true, G.mode == 1)) G.mode = 1;
                      if (UiButton((Rectangle) { SCREEN_W / 2 + 20.0f, 130, 280, 56 }, "Black", true, G.mode == 2)) G.mode = 2;
                  
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
                      float hitstop, slowT, fovKick, crackT, crackX, crackZ, stepT;
                      Vector3 tip, trail[10];
                      int ntr;
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
                  #define NPILL 6   /* 2 rune pillars + 4 corner statues */
                  static const float PILLARS[NPILL][2] = { { -4.5f, 0.0f }, { 4.5f, 0.0f }, { -9.9f, -9.9f }, { 9.9f, -9.9f }, { -9.9f, 9.9f }, { 9.9f, 9.9f } };
                  
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
                      for (int i = 0; i < NPILL; i++) {
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
                  
                  
                  /* ----- effects : particles, shock rings, floating damage numbers ----- */
                  #define MAX_PART 600
                  typedef struct { Vector3 p, v; float life, max, size, grow, grav, drag; Color c; int kind; bool on; } Part;
                  typedef struct { Vector3 p, axis; float t, max, r0, r1, ang; Color c; bool on; } FxRing;
                  typedef struct { Vector3 p; float val, life; Color c; bool on; } DmgNum;
                  
                  static Part PT[MAX_PART];
                  static FxRing RG[12];
                  static DmgNum DN[24];
                  
                  static Color Col(int r, int g, int b, int a)
                  {
                      Color c;
                      c.r = (unsigned char)r; c.g = (unsigned char)g; c.b = (unsigned char)b; c.a = (unsigned char)a;
                      return c;
                  }
                  static float RandF(float a, float b) { return a + (b - a) * (GetRandomValue(0, 1000) / 1000.0f); }
                  
                  static void AddPart(Vector3 p, Vector3 v, float life, float size, float grow, float grav, float drag, Color c, int kind)
                  {
                      static int cursor = 0;
                      for (int k = 0; k < MAX_PART; k++) {
                          int i = (cursor + k) % MAX_PART;
                          if (PT[i].on) continue;
                          Part* q = &PT[i];
                          q->p = p; q->v = v; q->life = q->max = life; q->size = size; q->grow = grow;
                          q->grav = grav; q->drag = drag; q->c = c; q->kind = kind; q->on = true;
                          cursor = i + 1;
                          return;
                      }
                  }
                  
                  /* kind 0 = glowing orb, 1 = spark (line), 2 = smoke puff */
                  static void Burst(Vector3 p, int n, float spd, float life, float size, float grav, Color c, int kind)
                  {
                      for (int i = 0; i < n; i++) {
                          float a = RandF(0.0f, 2.0f * PI_F), z = RandF(-1.0f, 1.0f), r = sqrtf(1.0f - z * z);
                          float sp = spd * RandF(0.35f, 1.0f);
                          AddPart(p, V3(r * cosf(a) * sp, z * sp * 0.8f + sp * 0.25f, r * sinf(a) * sp),
                              life * RandF(0.6f, 1.0f), size * RandF(0.6f, 1.2f), kind == 2 ? 0.6f : 0.0f, grav,
                              kind == 2 ? 1.5f : 0.4f, c, kind);
                      }
                  }
                  
                  static void AddRing(Vector3 p, Vector3 axis, float ang, float r0, float r1, float t, Color c)
                  {
                      for (int i = 0; i < 12; i++) {
                          if (RG[i].on) continue;
                          RG[i].p = p; RG[i].axis = axis; RG[i].ang = ang; RG[i].r0 = r0; RG[i].r1 = r1;
                          RG[i].t = 0.0f; RG[i].max = t; RG[i].c = c; RG[i].on = true;
                          return;
                      }
                  }
                  
                  static void AddDmg(Vector3 p, float val, Color c)
                  {
                      for (int i = 0; i < 24; i++) {
                          if (DN[i].on) continue;
                          DN[i].p = V3(p.x + RandF(-0.3f, 0.3f), p.y, p.z + RandF(-0.3f, 0.3f));
                          DN[i].val = val; DN[i].life = 0.9f; DN[i].c = c; DN[i].on = true;
                          return;
                      }
                  }
                  
                  static void ClearFx(void)
                  {
                      memset(PT, 0, sizeof PT);
                      memset(RG, 0, sizeof RG);
                      memset(DN, 0, sizeof DN);
                  }
                  
                  static void UpdateFx(float dt)
                  {
                      for (int i = 0; i < MAX_PART; i++) {
                          Part* q = &PT[i];
                          if (!q->on) continue;
                          q->life -= dt;
                          if (q->life <= 0.0f) { q->on = false; continue; }
                          q->v.y -= q->grav * dt;
                          float k = 1.0f - q->drag * dt;
                          if (k < 0.0f) k = 0.0f;
                          q->v.x *= k; q->v.y *= k; q->v.z *= k;
                          q->p.x += q->v.x * dt; q->p.y += q->v.y * dt; q->p.z += q->v.z * dt;
                          q->size += q->grow * dt;
                          if (q->size < 0.01f) q->size = 0.01f;
                          if (q->kind == 1 && q->p.y < 0.03f) { q->p.y = 0.03f; q->v.y *= -0.3f; q->v.x *= 0.6f; q->v.z *= 0.6f; }
                      }
                      for (int i = 0; i < 12; i++) {
                          if (!RG[i].on) continue;
                          RG[i].t += dt;
                          if (RG[i].t >= RG[i].max) RG[i].on = false;
                      }
                      for (int i = 0; i < 24; i++) {
                          if (!DN[i].on) continue;
                          DN[i].life -= dt;
                          if (DN[i].life <= 0.0f) DN[i].on = false;
                      }
                  }
                  
                  static Vector3 TorchPos(int i)
                  {
                      int w = i / 4, k = i % 4;
                      float a = -9.0f + 6.0f * k;
                      switch (w) {
                      case 0:  return V3(a, 2.7f, -11.15f);
                      case 1:  return V3(a, 2.7f, 11.15f);
                      case 2:  return V3(-11.15f, 2.7f, a);
                      default: return V3(11.15f, 2.7f, a);
                      }
                  }
                  
                  /* continuous ambience + telegraph effects (runs with the simulation) */
                  static void AmbientFx(void)
                  {
                      if (GetRandomValue(0, 100) < 45) {                      /* torch embers */
                          Vector3 t = TorchPos(GetRandomValue(0, 15));
                          AddPart(V3(t.x + RandF(-0.05f, 0.05f), t.y + 0.25f, t.z + RandF(-0.05f, 0.05f)),
                              V3(RandF(-0.2f, 0.2f), RandF(0.6f, 1.4f), RandF(-0.2f, 0.2f)),
                              RandF(0.8f, 1.6f), 0.035f, -0.02f, 0.0f, 0.2f, Col(255, 170, 60, 255), 0);
                      }
                      if (GetRandomValue(0, 100) < 20)                        /* floating dust motes */
                          AddPart(V3(RandF(-11.0f, 11.0f), RandF(0.3f, 4.0f), RandF(-11.0f, 11.0f)),
                              V3(RandF(-0.1f, 0.1f), RandF(0.05f, 0.2f), RandF(-0.1f, 0.1f)),
                              RandF(2.5f, 4.5f), 0.03f, 0.0f, 0.0f, 0.0f, Col(255, 240, 200, 140), 0);
                      if (D.ultT > 0.0f && GetRandomValue(0, 100) < 85) {     /* overdrive aura */
                          float a = RandF(0.0f, 2.0f * PI_F);
                          AddPart(V3(D.px + cosf(a) * 1.3f, 0.1f, D.pz + sinf(a) * 1.3f), V3(0.0f, RandF(1.0f, 3.0f), 0.0f),
                              0.7f, 0.05f, -0.04f, 0.0f, 0.0f, Col(255, 200, 70, 255), 0);
                      }
                      if (D.es == E_WINDUP_SLAM && D.estun <= 0.0f && GetRandomValue(0, 100) < 85) {   /* slam telegraph */
                          float a = RandF(0.0f, 2.0f * PI_F), r = RandF(0.0f, 3.4f);
                          AddPart(V3(D.slamX + cosf(a) * r, 0.1f, D.slamZ + sinf(a) * r), V3(0.0f, RandF(1.5f, 3.5f), 0.0f),
                              0.7f, 0.07f, -0.05f, 0.0f, 0.0f, Col(255, 70, 40, 255), 0);
                      }
                      if (D.es == E_WINDUP_SWORD && D.estun <= 0.0f)           /* sword charging */
                          AddPart(V3(D.tip.x + RandF(-0.3f, 0.3f), D.tip.y + RandF(-0.3f, 0.3f), D.tip.z + RandF(-0.3f, 0.3f)),
                              V3(0.0f, 0.0f, 0.0f), 0.25f, 0.06f, -0.2f, 0.0f, 0.0f, Col(255, 70, 50, 255), 0);
                      if (!D.playerIsAttacker && GetRandomValue(0, 100) < 35) {   /* attacker aura on the enemy */
                          float a = RandF(0.0f, 2.0f * PI_F);
                          AddPart(V3(D.ex + cosf(a) * 0.8f, D.ey + 0.1f, D.ez + sinf(a) * 0.8f), V3(0.0f, RandF(0.8f, 2.0f), 0.0f),
                              0.8f, 0.045f, -0.04f, 0.0f, 0.0f, Col(255, 90, 50, 255), 0);
                      }
                      /* enemy sword trail */
                      if (D.strikeT > 0.0f || D.es == E_WINDUP_SWORD) {
                          for (int i = 9; i > 0; i--) D.trail[i] = D.trail[i - 1];
                          D.trail[0] = D.tip;
                          if (D.ntr < 10) D.ntr++;
                      }
                      else if (D.ntr > 0) D.ntr--;
                  }
                  
                  /* ----- damage helpers ----- */
                  static void HurtEnemy(float dmg)
                  {
                      if (D.estun > 0.0f) dmg *= 1.25f;
                      D.ehp -= dmg;
                      D.ehitT = 0.15f;
                      if (D.ultT <= 0.0f) { D.ult += dmg * 0.8f; if (D.ult > 100.0f) D.ult = 100.0f; }
                      Vector3 c = V3(D.ex, 1.3f + D.ey, D.ez);
                      Burst(c, dmg >= 18.0f ? 22 : 12, 6.5f, 0.45f, 0.05f, 12.0f, Col(255, 215, 110, 255), 1);
                      Burst(c, 6, 3.0f, 0.6f, 0.09f, 0.0f, D.ef.side > 0 ? Col(190, 230, 255, 255) : Col(255, 90, 90, 255), 0);
                      AddDmg(V3(D.ex, 2.2f + D.ey, D.ez), dmg, dmg >= 20.0f ? Col(255, 210, 70, 255) : Col(255, 255, 255, 255));
                      if (dmg >= 12.0f) { D.hitstop = 0.05f; D.fovKick = 4.0f; }
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
                      float pfx = sinf(D.yaw), pfz = cosf(D.yaw);
                      Vector3 c = V3(D.px + pfx * 1.2f, 1.5f, D.pz + pfz * 1.2f);
                      Burst(c, 46, 10.0f, 0.55f, 0.05f, 6.0f, Col(180, 235, 255, 255), 1);
                      Burst(c, 10, 2.5f, 0.5f, 0.12f, 0.0f, Col(255, 255, 255, 255), 0);
                      AddRing(c, V3(0.0f, 1.0f, 0.0f), D.yaw * 180.0f / PI_F, 0.2f, 2.4f, 0.4f, Col(150, 230, 255, 255));
                      D.slowT = 0.35f; D.hitstop = 0.09f; D.fovKick = 8.0f;
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
                      Burst(D.tip, 8, 5.0f, 0.2f, 0.05f, 0.0f, Col(255, 220, 100, 255), 1);
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
                          float want = (D.eAtkCd <= 0.0f) ? 1.9f : 3.4f;      /* ready -> close in, cooling down -> keep distance */
                          if (d > want) { mx += dx / d; mz += dz / d; }
                          else if (d < want - 0.7f) { mx -= dx / d * 0.6f; mz -= dz / d * 0.6f; }
                          float ml = sqrtf(mx * mx + mz * mz);
                          if (ml > 0.001f) { D.ex += mx / ml * spd * dt; D.ez += mz / ml * spd * dt; }
                          Collide(&D.ex, &D.ez, 0.5f);
                  
                          if (d < 2.9f && D.eAtkCd <= 0.0f) {
                              D.es = E_WINDUP_SWORD; D.esDur = D.esT = windup;
                          }
                          else if (d > 3.5f && D.slamCd <= 0.0f && GetRandomValue(0, 100) < 3) {
                              D.es = E_WINDUP_SLAM; D.esDur = D.esT = 1.15f;
                              D.slamX = D.px; D.slamZ = D.pz; D.slamSX = D.ex; D.slamSZ = D.ez;
                              D.slamCd = lvl == 1 ? 11.0f : (lvl == 2 ? 9.0f : 7.0f);
                          }
                          else if (d > 3.8f && D.eShotCd <= 0.0f && GetRandomValue(0, 100) < 4) {
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
                              if (d < 3.3f && D.iframes <= 0.0f) {
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
                              {
                                  Vector3 c = V3(D.slamX, 0.1f, D.slamZ);
                                  Burst(c, 34, 5.0f, 1.0f, 0.5f, -0.5f, Col(120, 100, 90, 160), 2);
                                  Burst(c, 40, 9.0f, 0.8f, 0.05f, 9.0f, Col(255, 150, 50, 255), 1);
                                  AddRing(c, V3(1.0f, 0.0f, 0.0f), 90.0f, 0.5f, 5.0f, 0.5f, Col(255, 140, 60, 255));
                                  AddRing(c, V3(1.0f, 0.0f, 0.0f), 90.0f, 0.3f, 3.4f, 0.4f, Col(255, 220, 120, 255));
                                  D.crackT = 1.6f; D.crackX = D.slamX; D.crackZ = D.slamZ;
                                  D.hitstop = 0.06f;
                              }
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
                      for (int i = 0; i < NPILL; i++)
                          for (int k = 0; k < 4; k++) {
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
                      {
                          bool hitE = (best > 0.0f && (tWall < 0.0f || best < tWall));
                          if (!hitE) {
                              Burst(end, 8, 4.0f, 0.35f, 0.04f, 8.0f, Col(255, 230, 150, 255), 1);
                              Burst(end, 3, 1.5f, 0.6f, 0.2f, 0.0f, Col(150, 140, 140, 120), 2);
                          }
                          Burst(D.trA, 4, 3.0f, 0.15f, 0.05f, 0.0f, Col(255, 220, 100, 255), 1);
                      }
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
                  
                      D.stepT -= dt;
                      if (ml > 0.0f && D.dashT <= 0.0f && D.stepT <= 0.0f) {
                          D.stepT = 0.28f;
                          Burst(V3(D.px, 0.05f, D.pz), 2, 1.2f, 0.5f, 0.12f, -0.1f, Col(200, 190, 180, 90), 2);
                      }
                      if (D.dashT > 0.0f) Burst(V3(D.px, 0.2f, D.pz), 1, 1.0f, 0.4f, 0.25f, 0.0f, Col(120, 220, 255, 120), 2);
                  
                      /* abilities */
                      if (IsKeyPressed(KEY_Q) && D.dashCdT <= 0.0f) {
                          float ddx = ml > 0.0f ? mx : fx, ddz = ml > 0.0f ? mz : fz;
                          D.dvx = ddx * 24.0f; D.dvz = ddz * 24.0f;
                          D.dashT = 0.17f; D.iframes = 0.25f; D.dashCdT = D.pf.dashCd;
                          Burst(V3(D.px, 0.1f, D.pz), 14, 3.0f, 0.5f, 0.3f, -0.2f, Col(190, 200, 210, 140), 2);
                      }
                      if (IsKeyPressed(KEY_E) && D.shockCdT <= 0.0f) {
                          D.shockCdT = 8.0f; D.shockFx = 0.45f; D.shake = 0.3f;
                          {
                              Vector3 c = V3(D.px, 0.1f, D.pz);
                              AddRing(c, V3(1.0f, 0.0f, 0.0f), 90.0f, 0.4f, 4.6f, 0.45f, Col(120, 225, 255, 255));
                              AddRing(c, V3(1.0f, 0.0f, 0.0f), 90.0f, 0.2f, 3.4f, 0.35f, Col(220, 250, 255, 255));
                              Burst(c, 50, 9.0f, 0.5f, 0.06f, 0.0f, Col(130, 225, 255, 255), 0);
                              Burst(c, 14, 4.0f, 0.8f, 0.4f, -0.3f, Col(190, 200, 210, 120), 2);
                              D.fovKick = 6.0f;
                          }
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
                          AddPart(V3(b->x, b->y, b->z), V3(0.0f, 0.0f, 0.0f), 0.25f, 0.07f, -0.2f, 0.0f, 0.0f, b->fromPlayer ? Col(120, 220, 255, 160) : Col(255, 150, 50, 160), 0);
                          b->x += b->vx * dt; b->y += b->vy * dt; b->z += b->vz * dt;
                          b->life -= dt;
                          if (b->life <= 0.0f || fabsf(b->x) > ARENA + 0.5f || fabsf(b->z) > ARENA + 0.5f || b->y < 0.0f || b->y > 4.0f) { b->active = false; continue; }
                          for (int k = 0; k < NPILL; k++)
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
                                      Burst(V3(b->x, b->y, b->z), 30, 8.0f, 0.5f, 0.05f, 6.0f, Col(180, 235, 255, 255), 1);
                                      AddRing(V3(b->x, b->y, b->z), V3(0.0f, 1.0f, 0.0f), D.yaw * 180.0f / PI_F, 0.2f, 1.6f, 0.35f, Col(150, 230, 255, 255));
                                      D.slowT = 0.25f; D.hitstop = 0.06f;
                                  }
                                  else {
                                      float dmg = b->dmg * (g == 1 ? 0.2f : 1.0f);
                                      if (g == 1) DuelMsg("BLOCK");
                                      HurtPlayer(dmg);
                                      Burst(V3(b->x, b->y, b->z), 8, 4.0f, 0.3f, 0.04f, 8.0f, Col(255, 180, 80, 255), 1);
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
                      {   /* the ATTACKER is much stronger, the defender slightly weaker */
                          FStats* att = D.playerIsAttacker ? &D.pf : &D.ef;
                          FStats* dfd = D.playerIsAttacker ? &D.ef : &D.pf;
                          att->maxHp *= 1.7f; att->swordMul *= 1.4f; att->gunMul *= 1.4f;
                          att->speedMul *= 1.12f; att->dashCd *= 0.65f;
                          dfd->maxHp *= 0.85f;
                      }
                      ClearFx();
                      if (D.playerIsAttacker) D.ult = 50.0f;
                      float hpMul = G.aiLevel == 1 ? 0.8f : (G.aiLevel == 2 ? 1.0f : 1.25f);
                      D.pmax = D.php = D.pf.maxHp;
                      D.emax = D.ehp = D.ef.maxHp * hpMul;
                  
                      D.px = 0.0f; D.pz = -8.0f; D.yaw = 0.0f; D.pitch = 0.0f;
                      D.ex = 0.0f; D.ez = 8.0f; D.eyaw = PI_F;
                      D.ammo = 12;
                      D.timeLeft = 90.0f;
                      D.eAtkCd = 1.5f; D.eShotCd = 2.0f; D.slamCd = 5.0f; D.strafeDir = 1.0f; D.strafeT = 1.0f;
                      D.skipLook = 3;
                      if (!D.playerIsAttacker) D.eAtkCd = 0.8f;
                  
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
                      snprintf(G.san[G.nply], sizeof G.san[G.nply], "%.8s(L)", base);
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
                      D.slowT = 1.4f; D.hitstop = 0.12f;
                      if (playerWon) {
                          Vector3 c = V3(D.ex, 1.2f + D.ey, D.ez);
                          Burst(c, 90, 11.0f, 1.2f, 0.07f, 9.0f, D.ef.side > 0 ? Col(240, 240, 255, 255) : Col(255, 70, 70, 255), 1);
                          Burst(c, 40, 5.0f, 1.0f, 0.12f, 1.0f, Col(255, 210, 90, 255), 0);
                          Burst(c, 20, 3.0f, 1.2f, 0.6f, -0.4f, Col(120, 110, 110, 150), 2);
                          AddRing(c, V3(1.0f, 0.0f, 0.0f), 90.0f, 0.5f, 6.0f, 0.7f, Col(255, 220, 120, 255));
                          D.es = E_IDLE; D.ey = 0.0f;
                      }
                      else {
                          D.shake = 0.5f;
                      }
                  }
                  
                  /* returns true when the duel is over and the scene has changed */
                  static bool UpdateDuel(float dt)
                  {
                      D.msgT -= dt; D.hurtT -= dt; D.shake -= dt * 1.5f; D.sparkT -= dt;
                      D.shockFx -= dt; D.trT -= dt; D.fireT -= dt;
                      D.crackT -= dt;
                      D.fovKick -= dt * 40.0f; if (D.fovKick < 0.0f) D.fovKick = 0.0f;
                      UpdateFx(dt);
                      if (D.phase != 2 || !D.playerWon) AmbientFx();
                  
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
                  
                  static void DrawEnemy(void);
                  /* ----- 3D drawing ----- */
                  static void Orb(Vector3 p, float r, Color c) { DrawSphereEx(p, r, 8, 8, c); }
                  
                  static float Hash(int a, int b)
                  {
                      unsigned int h = (unsigned int)a * 374761393u + (unsigned int)b * 668265263u;
                      h = (h ^ (h >> 13)) * 1274126177u;
                      return ((h ^ (h >> 16)) & 1023u) / 1023.0f;
                  }
                  
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
                      gCam.fovy = 75.0f + (D.dashT > 0.0f ? 10.0f : 0.0f) + D.fovKick;
                      gCam.projection = CAMERA_PERSPECTIVE;
                  }
                  
                  /* chess-piece statues in the corners */
                  static void DrawStatue(int type, float x, float z, Color c)
                  {
                      Color d = Col(c.r * 7 / 10, c.g * 7 / 10, c.b * 7 / 10, 255);
                      DrawCube(V3(x, 0.4f, z), 1.7f, 0.8f, 1.7f, d);
                      DrawCube(V3(x, 0.86f, z), 1.4f, 0.12f, 1.4f, c);
                      float y0 = 0.92f;
                      switch (type) {
                      case ROOK:
                          DrawCylinderEx(V3(x, y0, z), V3(x, y0 + 2.0f, z), 0.62f, 0.5f, 14, c);
                          DrawCylinderEx(V3(x, y0 + 2.0f, z), V3(x, y0 + 2.4f, z), 0.7f, 0.7f, 14, c);
                          for (int i = 0; i < 4; i++)
                              DrawCube(V3(x + ((i & 1) ? 0.5f : -0.5f), y0 + 2.6f, z + ((i & 2) ? 0.5f : -0.5f)), 0.3f, 0.35f, 0.3f, c);
                          break;
                      case KNIGHT:
                          DrawCylinderEx(V3(x, y0, z), V3(x, y0 + 1.4f, z), 0.6f, 0.38f, 14, c);
                          DrawCylinderEx(V3(x, y0 + 1.3f, z), V3(x + 0.45f, y0 + 2.2f, z), 0.3f, 0.24f, 10, c);
                          DrawCylinderEx(V3(x + 0.35f, y0 + 2.15f, z), V3(x + 1.0f, y0 + 2.0f, z), 0.24f, 0.14f, 10, c);
                          DrawCylinderEx(V3(x + 0.3f, y0 + 2.3f, z), V3(x + 0.25f, y0 + 2.75f, z), 0.08f, 0.0f, 6, c);
                          break;
                      case BISHOP:
                          DrawCylinderEx(V3(x, y0, z), V3(x, y0 + 2.2f, z), 0.6f, 0.22f, 14, c);
                          Orb(V3(x, y0 + 2.5f, z), 0.34f, c);
                          Orb(V3(x, y0 + 2.95f, z), 0.1f, Col(220, 190, 90, 255));
                          break;
                      default:
                          DrawCylinderEx(V3(x, y0, z), V3(x, y0 + 2.6f, z), 0.65f, 0.25f, 14, c);
                          for (int i = 0; i < 5; i++) {
                              float a = i * 2.0f * PI_F / 5.0f;
                              Orb(V3(x + cosf(a) * 0.3f, y0 + 2.85f, z + sinf(a) * 0.3f), 0.1f, Col(220, 190, 90, 255));
                          }
                          Orb(V3(x, y0 + 2.7f, z), 0.28f, c);
                          break;
                      }
                  }
                  
                  static void WallPart(int w, float y, float h, float grow, Color c)
                  {
                      bool ax = w < 2;
                      float sg = (w % 2 == 0) ? -1.0f : 1.0f;
                      if (ax) DrawCube(V3(0.0f, y, sg * 12.5f), 26.0f + grow * 2.0f, h, 1.0f + grow * 2.0f, c);
                      else    DrawCube(V3(sg * 12.5f, y, 0.0f), 1.0f + grow * 2.0f, h, 26.0f + grow * 2.0f, c);
                  }
                  
                  static void DrawDuelWorld(void)
                  {
                      float t = (float)GetTime();
                      ClearBackground(Col(8, 10, 22, 255));
                      BeginMode3D(gCam);
                  
                      /* ---- sky: stars, moon, distant mountains ---- */
                      for (int i = 0; i < 170; i++) {
                          float u = Hash(i, 1) * 2.0f * PI_F, v = 0.06f + Hash(i, 2) * 0.94f, r = sqrtf(1.0f - v * v);
                          float tw = 0.55f + 0.45f * sinf(t * 2.0f + i);
                          DrawPoint3D(V3(gCam.position.x + cosf(u) * r * 150.0f, gCam.position.y + v * 150.0f, gCam.position.z + sinf(u) * r * 150.0f),
                              Col(220, 230, 255, (int)(255 * tw)));
                      }
                      {
                          Vector3 m = V3(gCam.position.x - 50.0f, gCam.position.y + 80.0f, gCam.position.z + 110.0f);
                          Orb(m, 11.0f, Col(240, 242, 255, 255));
                          Orb(m, 17.0f, Col(190, 205, 255, 40));
                          Orb(m, 26.0f, Col(150, 170, 255, 18));
                      }
                      for (int i = 0; i < 16; i++) {
                          float a = i * 2.0f * PI_F / 16.0f, h = 20.0f + Hash(i, 7) * 26.0f;
                          float px = cosf(a) * 120.0f, pz = sinf(a) * 120.0f;
                          DrawCylinderEx(V3(px, -3.0f, pz), V3(px, h, pz), 30.0f, 0.0f, 9, Col(14, 17, 34, 255));
                      }
                  
                      /* ---- foundation + marble floor ---- */
                      DrawCube(V3(0.0f, -0.8f, 0.0f), 28.0f, 1.2f, 28.0f, Col(30, 26, 34, 255));
                      DrawCube(V3(0.0f, -0.13f, 0.0f), 24.0f, 0.14f, 24.0f, Col(22, 18, 24, 255));
                      for (int r = 0; r < 8; r++)
                          for (int c = 0; c < 8; c++) {
                              bool light = ((r + c) & 1) == 0;
                              float j = Hash(r, c) * 14.0f;
                              Color tc = light ? Col((int)(222 - j), (int)(212 - j), (int)(196 - j), 255)
                                  : Col((int)(66 + j * 0.6f), (int)(62 + j * 0.6f), (int)(84 + j * 0.6f), 255);
                              DrawCube(V3(-10.5f + c * 3.0f, -0.07f, -10.5f + r * 3.0f), 2.94f, 0.14f, 2.94f, tc);
                          }
                      Color gold = Col(205, 160, 60, 255);
                      DrawCube(V3(0.0f, 0.005f, -11.8f), 23.6f, 0.02f, 0.18f, gold);
                      DrawCube(V3(0.0f, 0.005f, 11.8f), 23.6f, 0.02f, 0.18f, gold);
                      DrawCube(V3(-11.8f, 0.005f, 0.0f), 0.18f, 0.02f, 23.6f, gold);
                      DrawCube(V3(11.8f, 0.005f, 0.0f), 0.18f, 0.02f, 23.6f, gold);
                      for (int k = 1; k <= 3; k++) DrawCircle3D(V3(0.0f, 0.03f, 0.0f), 1.1f * k, V3(1.0f, 0.0f, 0.0f), 90.0f, gold);
                      for (int i = 0; i < 8; i++) {
                          float a = i * PI_F / 4.0f + t * 0.1f;
                          DrawLine3D(V3(cosf(a) * 1.1f, 0.03f, sinf(a) * 1.1f), V3(cosf(a) * 3.3f, 0.03f, sinf(a) * 3.3f), gold);
                      }
                  
                      /* ---- walls, columns, torches, banners ---- */
                      Color stone = Col(86, 78, 84, 255), stoneDk = Col(60, 54, 62, 255), stoneLt = Col(116, 108, 114, 255);
                      for (int w = 0; w < 4; w++) {
                          bool ax = w < 2;
                          float sg = (w % 2 == 0) ? -1.0f : 1.0f;
                          WallPart(w, 0.35f, 0.7f, 0.08f, stoneDk);
                          WallPart(w, 2.5f, 3.4f, 0.0f, stone);
                          for (int k = 0; k < 7; k++) WallPart(w, 0.95f + k * 0.5f, 0.04f, 0.015f, stoneDk);
                          WallPart(w, 4.35f, 0.35f, 0.12f, stoneLt);
                          for (int i = -6; i <= 6; i++) {
                              if (ax) DrawCube(V3(i * 2.0f, 5.0f, sg * 12.5f), 1.0f, 0.9f, 1.2f, stone);
                              else    DrawCube(V3(sg * 12.5f, 5.0f, i * 2.0f), 1.2f, 0.9f, 1.0f, stone);
                          }
                          for (int k = 0; k < 4; k++) {
                              float a = -9.0f + 6.0f * k;
                              float cx = ax ? a : sg * 11.6f, cz = ax ? sg * 11.6f : a;
                              DrawCube(V3(cx, 0.45f, cz), 1.0f, 0.5f, 1.0f, stoneDk);
                              DrawCylinder(V3(cx, 0.7f, cz), 0.42f, 0.42f, 3.6f, 10, stoneLt);
                              DrawCube(V3(cx, 4.4f, cz), 1.05f, 0.32f, 1.05f, stoneDk);
                              Vector3 tp = TorchPos(w * 4 + k);
                              DrawCylinderEx(V3(cx, tp.y - 0.2f, cz), V3(tp.x, tp.y - 0.2f, tp.z), 0.05f, 0.05f, 6, Col(60, 44, 36, 255));
                              DrawCylinderEx(V3(tp.x, tp.y - 0.35f, tp.z), V3(tp.x, tp.y - 0.02f, tp.z), 0.05f, 0.1f, 6, Col(60, 44, 36, 255));
                              float fl = sinf(t * 11.0f + (w * 4 + k) * 1.7f) * 0.04f + sinf(t * 7.3f + k) * 0.03f;
                              Orb(V3(tp.x, tp.y + 0.12f + fl, tp.z), 0.16f, Col(255, 120, 30, 255));
                              Orb(V3(tp.x, tp.y + 0.18f + fl, tp.z), 0.10f, Col(255, 215, 100, 255));
                              Orb(V3(tp.x, tp.y + 0.20f, tp.z), 0.26f, Col(255, 140, 40, 40));
                              Orb(V3(tp.x, tp.y + 0.20f, tp.z), 0.55f, Col(255, 130, 40, 16));
                              float px = ax ? tp.x : tp.x - sg * 2.0f, pz = ax ? tp.z - sg * 2.0f : tp.z;
                              DrawCylinder(V3(px, 0.012f, pz), 2.4f, 2.4f, 0.01f, 18, Col(255, 150, 60, 14));
                          }
                          for (int k = 0; k < 3; k++) {
                              float a = -6.0f + 6.0f * k;
                              bool whiteBanner = ((k + w) & 1) == 0;
                              Color bc = whiteBanner ? Col(214, 208, 196, 255) : Col(110, 22, 34, 255);
                              Color tr = whiteBanner ? Col(190, 150, 60, 255) : Col(30, 24, 30, 255);
                              float bx = ax ? a : sg * 11.95f, bz = ax ? sg * 11.95f : a;
                              if (ax) {
                                  DrawCube(V3(bx, 2.35f, bz), 1.5f, 2.4f, 0.05f, bc);
                                  DrawCube(V3(bx, 3.6f, bz - sg * 0.03f), 1.7f, 0.1f, 0.1f, tr);
                                  DrawCube(V3(bx, 1.1f, bz - sg * 0.03f), 1.5f, 0.08f, 0.06f, tr);
                                  Orb(V3(bx, 2.4f, bz - sg * 0.05f), 0.22f, Col(205, 160, 60, 255));
                              }
                              else {
                                  DrawCube(V3(bx, 2.35f, bz), 0.05f, 2.4f, 1.5f, bc);
                                  DrawCube(V3(bx - sg * 0.03f, 3.6f, bz), 0.1f, 0.1f, 1.7f, tr);
                                  DrawCube(V3(bx - sg * 0.03f, 1.1f, bz), 0.06f, 0.08f, 1.5f, tr);
                                  Orb(V3(bx - sg * 0.05f, 2.4f, bz), 0.22f, Col(205, 160, 60, 255));
                              }
                          }
                      }
                  
                      /* ---- corner statues ---- */
                      static const int stype[4] = { ROOK, KNIGHT, BISHOP, QUEEN };
                      for (int i = 0; i < 4; i++) DrawStatue(stype[i], PILLARS[2 + i][0], PILLARS[2 + i][1], Col(138, 136, 150, 255));
                  
                      /* ---- rune pillars with crystals ---- */
                      for (int i = 0; i < 2; i++) {
                          float x = PILLARS[i][0], z = PILLARS[i][1];
                          DrawCube(V3(x, 0.15f, z), 2.2f, 0.3f, 2.2f, stoneDk);
                          DrawCube(V3(x, 0.45f, z), 1.8f, 0.3f, 1.8f, stone);
                          DrawCylinder(V3(x, 0.6f, z), 0.85f, 0.8f, 2.7f, 16, stoneLt);
                          for (int k = 0; k < 4; k++) DrawCylinder(V3(x, 0.9f + k * 0.65f, z), 0.9f, 0.9f, 0.1f, 16, stone);
                          DrawCube(V3(x, 3.4f, z), 1.9f, 0.3f, 1.9f, stone);
                          DrawCube(V3(x, 3.65f, z), 2.3f, 0.25f, 2.3f, stoneDk);
                          float pu = 0.5f + 0.5f * sinf(t * 2.0f + i * 2.0f);
                          DrawCylinder(V3(x, 1.5f, z), 0.93f, 0.93f, 0.08f, 16, Col(90, 220, 255, (int)(90 + 140 * pu)));
                          float cy = 4.3f + 0.12f * sinf(t * 2.0f + i);
                          Orb(V3(x, cy, z), 0.3f, Col(130, 235, 255, 255));
                          Orb(V3(x, cy, z), 0.6f, Col(110, 220, 255, 50));
                          Orb(V3(x, cy, z), 1.0f, Col(100, 200, 255, 20));
                          DrawCircle3D(V3(x, 0.04f, z), 1.4f + 0.1f * pu, V3(1.0f, 0.0f, 0.0f), 90.0f, Col(90, 220, 255, (int)(60 + 120 * pu)));
                      }
                  
                      /* ---- rotating rune ring overhead ---- */
                      for (int k = 0; k < 3; k++) DrawCircle3D(V3(0.0f, 9.0f + k * 0.05f, 0.0f), 10.0f - k * 0.1f, V3(1.0f, 0.0f, 0.0f), 90.0f, Col(120, 200, 255, 90));
                      for (int i = 0; i < 12; i++) {
                          float a = i * PI_F / 6.0f + t * 0.25f;
                          Orb(V3(cosf(a) * 10.0f, 9.0f, sinf(a) * 10.0f), 0.18f, Col(150, 220, 255, 255));
                          Orb(V3(cosf(a) * 10.0f, 9.0f, sinf(a) * 10.0f), 0.45f, Col(120, 200, 255, 40));
                      }
                  
                      /* ---- cracks from the slam ---- */
                      if (D.crackT > 0.0f) {
                          float f = ClampF(D.crackT / 1.6f, 0.0f, 1.0f);
                          for (int i = 0; i < 10; i++) {
                              float a = i * 2.0f * PI_F / 10.0f + Hash(i, 3), l = 1.6f + Hash(i, 4) * 2.4f;
                              Vector3 p0 = V3(D.crackX, 0.03f, D.crackZ);
                              Vector3 p1 = V3(D.crackX + cosf(a) * l * 0.55f, 0.03f, D.crackZ + sinf(a) * l * 0.55f);
                              Vector3 p2 = V3(D.crackX + cosf(a + 0.25f) * l, 0.03f, D.crackZ + sinf(a + 0.25f) * l);
                              DrawLine3D(p0, p1, Col(255, 120, 40, (int)(255 * f)));
                              DrawLine3D(p1, p2, Col(255, 80, 30, (int)(255 * f)));
                          }
                      }
                  
                      /* ---- slam telegraph ---- */
                      if (D.es == E_WINDUP_SLAM && D.estun <= 0.0f) {
                          float pulse = 0.5f + 0.5f * sinf(t * 14.0f);
                          for (int k = 1; k <= 3; k++)
                              DrawCircle3D(V3(D.slamX, 0.05f, D.slamZ), 3.4f * k / 3.0f, V3(1.0f, 0.0f, 0.0f), 90.0f,
                                  Col(255, 50, 40, (int)(120 + 120 * pulse)));
                          DrawCylinder(V3(D.slamX, 0.02f, D.slamZ), 3.4f, 3.4f, 0.01f, 24, Col(255, 40, 30, (int)(30 + 40 * pulse)));
                      }
                  
                      if (!(D.phase == 2 && D.playerWon)) DrawEnemy();
                  
                      /* enemy sword trail */
                      for (int i = 0; i + 1 < D.ntr; i++) {
                          float f = 1.0f - (float)i / 10.0f;
                          DrawLine3D(D.trail[i], D.trail[i + 1], Col(255, 90, 70, (int)(255 * f)));
                          DrawLine3D(V3(D.trail[i].x, D.trail[i].y + 0.05f, D.trail[i].z), V3(D.trail[i + 1].x, D.trail[i + 1].y + 0.05f, D.trail[i + 1].z), Col(255, 200, 150, (int)(200 * f)));
                      }
                  
                      /* bullets */
                      for (int i = 0; i < MAX_BULLETS; i++) {
                          Bullet* b = &D.bl[i];
                          if (!b->active) continue;
                          Color c = b->fromPlayer ? Col(120, 220, 255, 255) : Col(255, 140, 40, 255);
                          Orb(V3(b->x, b->y, b->z), 0.10f, c);
                          Orb(V3(b->x, b->y, b->z), 0.22f, Col(c.r, c.g, c.b, 60));
                          DrawLine3D(V3(b->x, b->y, b->z), V3(b->x - b->vx * 0.05f, b->y - b->vy * 0.05f, b->z - b->vz * 0.05f), c);
                      }
                      if (D.trT > 0.0f) {
                          DrawLine3D(D.trA, D.trB, Col(255, 235, 130, 255));
                          DrawLine3D(V3(D.trA.x, D.trA.y + 0.02f, D.trA.z), V3(D.trB.x, D.trB.y + 0.02f, D.trB.z), Col(255, 200, 80, 140));
                      }
                  
                      /* particles and rings last (transparent) */
                      for (int i = 0; i < MAX_PART; i++) {
                          Part* q = &PT[i];
                          if (!q->on) continue;
                          float f = ClampF(q->life / q->max, 0.0f, 1.0f);
                          Color c = q->c;
                          c.a = (unsigned char)(c.a * ClampF(f * 1.5f, 0.0f, 1.0f));
                          if (q->kind == 1) {
                              DrawLine3D(q->p, V3(q->p.x - q->v.x * 0.04f, q->p.y - q->v.y * 0.04f, q->p.z - q->v.z * 0.04f), c);
                              DrawSphereEx(q->p, q->size * 0.5f, 4, 4, c);
                          }
                          else {
                              DrawSphereEx(q->p, q->size, 5, 5, c);
                          }
                      }
                      for (int i = 0; i < 12; i++) {
                          if (!RG[i].on) continue;
                          float f = RG[i].t / RG[i].max;
                          float r = RG[i].r0 + (RG[i].r1 - RG[i].r0) * (1.0f - (1.0f - f) * (1.0f - f));
                          Color c = RG[i].c;
                          c.a = (unsigned char)(c.a * (1.0f - f));
                          DrawCircle3D(RG[i].p, r, RG[i].axis, RG[i].ang, c);
                          DrawCircle3D(RG[i].p, r * 0.95f, RG[i].axis, RG[i].ang, c);
                          DrawCircle3D(RG[i].p, r * 1.05f, RG[i].axis, RG[i].ang, c);
                      }
                      EndMode3D();
                  }
                  
                  /* ---- the fighter model ---- */
                  static void DrawEnemy(void)
                  {
                      static const float SC[7] = { 1.0f, 0.92f, 1.0f, 1.03f, 1.12f, 1.08f, 1.18f };
                      float t = (float)GetTime();
                      float sc = SC[D.ef.piece];
                      float x = D.ex, y = D.ey, z = D.ez;
                      float fx = sinf(D.eyaw), fz = cosf(D.eyaw);
                      float rx = -cosf(D.eyaw), rz = sinf(D.eyaw);
                      bool white = D.ef.side > 0;
                      bool eAtk = !D.playerIsAttacker;
                      Color armor = white ? Col(228, 228, 236, 255) : Col(58, 58, 72, 255);
                      Color trim = white ? Col(205, 165, 60, 255) : Col(175, 30, 46, 255);
                      Color dark = white ? Col(150, 150, 162, 255) : Col(28, 28, 36, 255);
                      Color cape = white ? Col(190, 190, 205, 255) : Col(110, 20, 32, 255);
                      Color glow = white ? Col(120, 220, 255, 255) : Col(255, 50, 50, 255);
                      if (D.ehitT > 0.0f) armor = Col(255, 120, 110, 255);
                      if (D.estun > 0.0f && (((int)(t * 12.0f)) & 1)) armor = Col(130, 170, 255, 255);
                  
                      /* shadow + aura */
                      DrawCylinder(V3(x, 0.01f, z), 0.6f, 0.6f, 0.02f, 16, Col(0, 0, 0, 100));
                      float ap = 0.5f + 0.5f * sinf(t * 4.0f);
                      DrawCircle3D(V3(x, 0.04f, z), 0.9f + 0.08f * ap, V3(1.0f, 0.0f, 0.0f), 90.0f, Col(glow.r, glow.g, glow.b, 160));
                      if (eAtk) {
                          DrawCircle3D(V3(x, 0.05f, z), 1.25f + 0.1f * ap, V3(1.0f, 0.0f, 0.0f), 90.0f, Col(255, 90, 40, 200));
                          DrawCircle3D(V3(x, 0.05f, z), 1.35f + 0.1f * ap, V3(1.0f, 0.0f, 0.0f), 90.0f, Col(255, 190, 70, 140));
                      }
                  
                      /* legs + boots */
                      float sw = sinf(t * 7.0f) * ((D.es == E_IDLE && D.estun <= 0.0f) ? 0.18f : 0.0f);
                      for (int s = -1; s <= 1; s += 2) {
                          float o = s * 0.17f, sw2 = sw * s;
                          Vector3 foot = V3(x + rx * o + fx * sw2, y + 0.1f, z + rz * o + fz * sw2);
                          DrawCylinderEx(V3(foot.x, y, foot.z), V3(x + rx * o, y + 0.9f * sc, z + rz * o), 0.11f, 0.14f, 8, dark);
                          Orb(V3(foot.x + fx * 0.06f, foot.y, foot.z + fz * 0.06f), 0.15f, trim);
                          Orb(V3(x + rx * o + fx * sw2 * 0.5f, y + 0.5f * sc, z + rz * o + fz * sw2 * 0.5f), 0.12f, armor);
                      }
                  
                      /* cape */
                      float cs = sinf(t * 3.0f) * 0.08f + (D.es == E_WINDUP_SLAM ? 0.3f : 0.0f);
                      DrawCylinderEx(V3(x - fx * 0.18f, y + 1.55f * sc, z - fz * 0.18f), V3(x - fx * (0.38f + cs), y + 0.2f, z - fz * (0.38f + cs)), 0.30f * sc, 0.46f * sc, 12, cape);
                  
                      /* torso, belt, chest gem */
                      DrawCylinderEx(V3(x, y + 0.85f * sc, z), V3(x, y + 1.55f * sc, z), 0.30f * sc, 0.36f * sc, 12, armor);
                      DrawCylinderEx(V3(x, y + 0.85f * sc, z), V3(x, y + 0.98f * sc, z), 0.32f * sc, 0.32f * sc, 12, trim);
                      DrawCylinderEx(V3(x, y + 1.40f * sc, z), V3(x, y + 1.46f * sc, z), 0.37f * sc, 0.37f * sc, 12, trim);
                      Orb(V3(x + fx * 0.30f * sc, y + 1.2f * sc, z + fz * 0.30f * sc), 0.08f, glow);
                      Orb(V3(x + fx * 0.30f * sc, y + 1.2f * sc, z + fz * 0.30f * sc), 0.16f, Col(glow.r, glow.g, glow.b, 70));
                  
                      /* pauldrons */
                      Vector3 sL = V3(x - rx * 0.40f * sc, y + 1.45f * sc, z - rz * 0.40f * sc);
                      Vector3 sR = V3(x + rx * 0.40f * sc, y + 1.45f * sc, z + rz * 0.40f * sc);
                      Orb(sL, 0.2f * sc, armor);
                      Orb(sR, 0.2f * sc, armor);
                      Orb(V3(sL.x, sL.y + 0.1f, sL.z), 0.08f, trim);
                      Orb(V3(sR.x, sR.y + 0.1f, sR.z), 0.08f, trim);
                  
                      /* head, visor, glowing eyes, crest */
                      Vector3 head = V3(x, y + 1.80f * sc, z);
                      DrawSphereEx(head, 0.21f, 12, 12, armor);
                      Orb(V3(head.x + fx * 0.14f, head.y, head.z + fz * 0.14f), 0.10f, dark);
                      for (int s = -1; s <= 1; s += 2) {
                          Orb(V3(head.x + fx * 0.19f + rx * 0.06f * s, head.y + 0.01f, head.z + fz * 0.19f + rz * 0.06f * s), 0.03f, glow);
                          Orb(V3(head.x + fx * 0.19f + rx * 0.06f * s, head.y + 0.01f, head.z + fz * 0.19f + rz * 0.06f * s), 0.07f, Col(glow.r, glow.g, glow.b, 70));
                      }
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
                              Orb(V3(head.x + cosf(a) * 0.17f, head.y + 0.28f, head.z + sinf(a) * 0.17f), 0.07f, trim);
                          }
                          break;
                      case KING:
                          DrawCylinderEx(V3(head.x, head.y + 0.18f, head.z), V3(head.x, head.y + 0.62f, head.z), 0.05f, 0.05f, 6, trim);
                          DrawCylinderEx(V3(head.x - rx * 0.16f, head.y + 0.46f, head.z - rz * 0.16f), V3(head.x + rx * 0.16f, head.y + 0.46f, head.z + rz * 0.16f), 0.05f, 0.05f, 6, trim);
                          break;
                      default: break;
                      }
                      if (D.estun > 0.0f)                                       /* dizzy stars */
                          for (int i = 0; i < 3; i++) {
                              float a = t * 6.0f + i * 2.094f;
                              Orb(V3(head.x + cosf(a) * 0.35f, head.y + 0.4f, head.z + sinf(a) * 0.35f), 0.06f, Col(255, 230, 90, 255));
                          }
                  
                      /* arms + weapons */
                      Vector3 handL = V3(sL.x + fx * 0.15f, sL.y - 0.55f, sL.z + fz * 0.15f);
                      DrawCylinderEx(sL, handL, 0.08f, 0.07f, 8, armor);
                      Orb(handL, 0.09f, dark);
                  
                      Vector3 hand, tip;
                      bool gun = (D.es == E_WINDUP_SHOT) || D.gunT > 0.0f;
                      float p = (D.esDur > 0.0f) ? 1.0f - D.esT / D.esDur : 0.0f;
                      p = ClampF(p, 0.0f, 1.0f);
                      Color blade = Col(225, 228, 238, 255);
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
                          blade = Col((int)(225 + 30 * p), (int)(228 - 150 * p), (int)(238 - 170 * p), 255);
                      }
                      else if (D.es == E_WINDUP_SLAM) {
                          hand = V3(sR.x, sR.y + 0.7f, sR.z);
                          tip = V3(hand.x, hand.y + 0.9f, hand.z);
                          blade = Col(255, 140, 80, 255);
                      }
                      else if (D.estun > 0.0f) {
                          hand = V3(sR.x + fx * 0.1f, sR.y - 0.5f, sR.z + fz * 0.1f);
                          tip = V3(hand.x + fx * 0.6f, hand.y - 0.2f, hand.z + fz * 0.6f);
                      }
                      else {
                          hand = V3(sR.x + fx * 0.3f, sR.y - 0.3f, sR.z + fz * 0.3f);
                          tip = V3(hand.x + fx * 0.2f, hand.y + 0.85f, hand.z + fz * 0.2f);
                      }
                      D.tip = tip;
                      DrawCylinderEx(sR, hand, 0.08f, 0.07f, 8, armor);
                      Orb(hand, 0.09f, dark);
                      if (gun) {
                          DrawCylinderEx(hand, tip, 0.05f, 0.04f, 6, Col(40, 40, 46, 255));
                          DrawCylinderEx(V3(hand.x, hand.y - 0.02f, hand.z), V3(hand.x, hand.y - 0.2f, hand.z), 0.04f, 0.04f, 6, Col(70, 48, 36, 255));
                          if (D.gunT > 0.4f) {
                              Orb(V3(tip.x + fx * 0.1f, tip.y, tip.z + fz * 0.1f), 0.14f, Col(255, 225, 100, 255));
                              Orb(V3(tip.x + fx * 0.1f, tip.y, tip.z + fz * 0.1f), 0.3f, Col(255, 190, 60, 90));
                          }
                      }
                      else {
                          DrawCylinderEx(hand, tip, 0.045f, 0.012f, 6, blade);
                          DrawCylinderEx(V3(hand.x - rx * 0.13f, hand.y, hand.z - rz * 0.13f), V3(hand.x + rx * 0.13f, hand.y, hand.z + rz * 0.13f), 0.025f, 0.025f, 6, trim);
                          if (D.es == E_WINDUP_SWORD && p > 0.5f) {
                              Orb(tip, 0.12f, Col(255, 60, 40, 220));
                              Orb(tip, 0.32f, Col(255, 60, 40, 60));
                          }
                          if (D.es == E_WINDUP_SLAM) Orb(tip, 0.3f, Col(255, 150, 60, 90));
                      }
                  }
                  
                  /* ----- 2D overlay : weapons, HUD ----- */
                  static void DrawBladeScreen(Vector2 base, float angDeg, float len, unsigned char a, bool ultGlow)
                  {
                      float r = angDeg * PI_F / 180.0f;
                      Vector2 tip; tip.x = base.x + sinf(r) * len; tip.y = base.y - cosf(r) * len;
                      if (ultGlow) DrawLineEx(base, tip, 46.0f, Col(255, 190, 60, a / 5));
                      DrawLineEx(base, tip, 30.0f, Col(110, 122, 148, a));
                      DrawLineEx(base, tip, 22.0f, Col(170, 182, 205, a));
                      DrawLineEx(base, tip, 10.0f, Col(240, 244, 252, a));
                      Vector2 mid; mid.x = base.x + sinf(r) * len * 0.5f; mid.y = base.y - cosf(r) * len * 0.5f;
                      DrawLineEx(base, tip, 3.0f, Col(120, 135, 160, a));
                      Vector2 g; g.x = base.x + sinf(r) * len * 0.2f; g.y = base.y - cosf(r) * len * 0.2f;
                      Vector2 perp; perp.x = cosf(r); perp.y = sinf(r);
                      Vector2 ga = { g.x - perp.x * 58.0f, g.y - perp.y * 58.0f }, gb = { g.x + perp.x * 58.0f, g.y + perp.y * 58.0f };
                      DrawLineEx(ga, gb, 16.0f, Col(150, 112, 40, a));
                      DrawLineEx(ga, gb, 9.0f, Col(230, 190, 80, a));
                      DrawCircleV(ga, 9.0f, Col(230, 190, 80, a));
                      DrawCircleV(gb, 9.0f, Col(230, 190, 80, a));
                      (void)mid;
                  }
                  
                  static void DrawSwordViewmodel(void)
                  {
                      float t = (float)GetTime();
                      Vector2 base = { SCREEN_W * 0.80f, SCREEN_H + 30.0f };
                      float ang = 16.0f + sinf(t * 2.0f) * 1.5f, len = 470.0f;
                      bool ug = D.ultT > 0.0f;
                      if (D.guarding) { base.x = SCREEN_W * 0.60f; ang = -38.0f; len = 520.0f; }
                      float prog = -1.0f;
                      if (D.swingT > 0.0f) prog = 1.0f - D.swingT / 0.38f;
                      if (prog >= 0.0f) {
                          for (int k = 4; k >= 1; k--) {
                              float pe = ClampF(prog - 0.05f * k, 0.0f, 1.0f);
                              float e2 = pe * pe * (3.0f - 2.0f * pe);
                              Vector2 b2 = { base.x - 80.0f * e2, base.y };
                              DrawBladeScreen(b2, 40.0f - 120.0f * e2, len, (unsigned char)(26 + 8 * (5 - k)), ug);
                          }
                          float e = prog * prog * (3.0f - 2.0f * prog);
                          base.x -= 80.0f * e;
                          ang = 40.0f - 120.0f * e;
                      }
                      DrawBladeScreen(base, ang, len, 255, ug);
                      DrawRectangle((int)base.x - 38, (int)base.y - 60, 76, 90, Col(36, 30, 34, 255));     /* gauntlet */
                      DrawRectangle((int)base.x - 38, (int)base.y - 60, 76, 12, Col(90, 76, 64, 255));
                      if (D.guarding && D.parryT > 0.0f)
                          DrawCircleV((Vector2) { SCREEN_W * 0.5f, SCREEN_H * 0.45f }, 70.0f + D.parryT * 120.0f, Col(120, 220, 255, 70));
                  }
                  
                  static void DrawGunViewmodel(void)
                  {
                      float rec = D.fireT > 0.0f ? 18.0f * (D.fireT / 0.1f) : 0.0f;
                      float low = 0.0f;
                      if (D.reloadT > 0.0f) low = 200.0f * sinf(PI_F * (1.0f - D.reloadT / 1.2f));
                      float ox = SCREEN_W - 480.0f, oy = SCREEN_H - 200.0f + rec + low;
                      DrawRectangle((int)ox + 100, (int)oy + 18, 130, 60, Col(48, 50, 58, 255));
                      DrawRectangle((int)ox + 100, (int)oy + 18, 130, 10, Col(110, 114, 130, 255));
                      DrawRectangle((int)ox + 100, (int)oy + 66, 130, 12, Col(30, 32, 38, 255));
                      for (int i = 0; i < 5; i++) DrawRectangle((int)ox + 190 + i * 7, (int)oy + 28, 3, 36, Col(26, 28, 34, 255));
                      DrawRectangle((int)ox - 20, (int)oy + 30, 130, 30, Col(34, 36, 42, 255));
                      DrawRectangle((int)ox - 20, (int)oy + 30, 130, 6, Col(80, 84, 98, 255));
                      DrawRectangle((int)ox - 26, (int)oy + 22, 12, 12, Col(30, 30, 34, 255));
                      DrawRectangle((int)ox + 214, (int)oy + 8, 18, 14, Col(30, 30, 34, 255));
                      DrawRectangle((int)ox + 180, (int)oy + 70, 56, 140, Col(74, 50, 38, 255));
                      DrawRectangle((int)ox + 180, (int)oy + 70, 10, 140, Col(100, 70, 52, 255));
                      DrawRectangle((int)ox + 120, (int)oy + 78, 60, 8, Col(28, 28, 32, 255));
                      DrawText(TextFormat("%d", D.ammo), (int)ox + 140, (int)oy + 36, 22, D.ammo <= 3 ? Col(255, 90, 70, 255) : Col(120, 235, 255, 255));
                      if (D.fireT > 0.06f) {
                          DrawCircleV((Vector2) { ox - 30.0f, oy + 44.0f }, 40.0f, Col(255, 190, 60, 120));
                          DrawCircleV((Vector2) { ox - 30.0f, oy + 44.0f }, 30.0f, Col(255, 225, 100, 230));
                          DrawCircleV((Vector2) { ox - 30.0f, oy + 44.0f }, 16.0f, Col(255, 255, 230, 255));
                      }
                  }
                  
                  static void DrawBar(float x, float y, float w, float h, float frac, Color c)
                  {
                      DrawRectangle((int)x - 2, (int)y - 2, (int)w + 4, (int)h + 4, Col(0, 0, 0, 200));
                      DrawRectangle((int)x, (int)y, (int)w, (int)h, Col(40, 18, 20, 255));
                      int fw = (int)(w * ClampF(frac, 0.0f, 1.0f));
                      DrawRectangleGradientV((int)x, (int)y, fw, (int)h, Col(c.r, c.g, c.b, 255), Col(c.r / 2, c.g / 2, c.b / 2, 255));
                      DrawRectangle((int)x, (int)y, fw, 3, Col(255, 255, 255, 70));
                  }
                  
                  static void DrawAbility(float x, float y, const char* key, const char* name, float cdLeft, float cdMax, Color c)
                  {
                      Rectangle r = { x, y, 74.0f, 74.0f };
                      DrawRectangleRounded(r, 0.18f, 6, Col(20, 22, 32, 225));
                      if (cdLeft > 0.0f && cdMax > 0.0f) {
                          float f = ClampF(cdLeft / cdMax, 0.0f, 1.0f);
                          DrawRectangle((int)x + 3, (int)(y + 3 + 68.0f * (1.0f - f)), 68, (int)(68.0f * f), Col(0, 0, 0, 170));
                          DrawTextC(TextFormat("%.1f", cdLeft), (int)x + 37, (int)y + 24, 20, LIGHTGRAY);
                      }
                      else {
                          DrawRectangleLinesEx(r, 3.0f, c);
                          DrawRectangle((int)x + 3, (int)y + 3, 68, 68, Col(c.r, c.g, c.b, 30));
                      }
                      DrawText(key, (int)x + 7, (int)y + 5, 20, c);
                      DrawTextC(name, (int)x + 37, (int)y + 54, 14, LIGHTGRAY);
                  }
                  
                  static void DrawMinimap(void)
                  {
                      float cx = 90.0f, cy = 190.0f, k = 5.0f;
                      DrawRectangle((int)cx - 62, (int)cy - 62, 124, 124, Col(8, 10, 18, 190));
                      DrawRectangleLinesEx((Rectangle) { cx - 62, cy - 62, 124, 124 }, 2.0f, Col(120, 130, 170, 200));
                      for (int i = 0; i < NPILL; i++)
                          DrawCircleV((Vector2) { cx - PILLARS[i][0] * k, cy - PILLARS[i][1] * k }, PILLAR_R* k, i < 2 ? Col(90, 190, 230, 200) : Col(100, 100, 120, 200));
                      if (D.es == E_WINDUP_SLAM)
                          DrawCircleV((Vector2) { cx - D.slamX * k, cy - D.slamZ * k }, 3.4f * k, Col(255, 60, 40, 110));
                      Vector2 pe = { cx - D.ex * k, cy - D.ez * k };
                      DrawCircleV(pe, 5.0f, D.ef.side > 0 ? Col(240, 240, 240, 255) : Col(220, 60, 60, 255));
                      Vector2 pp = { cx - D.px * k, cy - D.pz * k };
                      DrawCircleV(pp, 5.0f, Col(110, 230, 120, 255));
                      Vector2 pf = { pp.x - sinf(D.yaw) * 14.0f, pp.y - cosf(D.yaw) * 14.0f };
                      DrawLineEx(pp, pf, 2.5f, Col(110, 230, 120, 255));
                  }
                  
                  static void DrawDuelHUD(void)
                  {
                      /* vignette */
                      DrawRectangleGradientH(0, 0, 220, SCREEN_H, Col(0, 0, 0, 120), Col(0, 0, 0, 0));
                      DrawRectangleGradientH(SCREEN_W - 220, 0, 220, SCREEN_H, Col(0, 0, 0, 0), Col(0, 0, 0, 120));
                      DrawRectangleGradientV(0, 0, SCREEN_W, 120, Col(0, 0, 0, 110), Col(0, 0, 0, 0));
                      DrawRectangleGradientV(0, SCREEN_H - 140, SCREEN_W, 140, Col(0, 0, 0, 0), Col(0, 0, 0, 130));
                  
                      if (D.hurtT > 0.0f)
                          DrawRectangle(0, 0, SCREEN_W, SCREEN_H, Col(255, 0, 0, (int)(100 * ClampF(D.hurtT / 0.35f, 0.0f, 1.0f))));
                      if (D.php < D.pmax * 0.3f && D.phase == 1)
                          DrawRectangle(0, 0, SCREEN_W, SCREEN_H, Col(255, 0, 0, (int)(20 + 18 * sinf((float)GetTime() * 6.0f))));
                      if (D.ultT > 0.0f)
                          DrawRectangleLinesEx((Rectangle) { 0, 0, SCREEN_W, SCREEN_H }, 8.0f, Col(255, 190, 60, 150));
                  
                      if (D.dashT > 0.0f)                                       /* speed lines */
                          for (int i = 0; i < 22; i++) {
                              float a = i * 2.0f * PI_F / 22.0f + RandF(-0.1f, 0.1f), r0 = RandF(260.0f, 340.0f), r1 = r0 + RandF(120.0f, 260.0f);
                              DrawLineEx((Vector2) { SCREEN_W / 2 + cosf(a) * r0, SCREEN_H / 2 + sinf(a) * r0 },
                                  (Vector2) {
                                  SCREEN_W / 2 + cosf(a) * r1, SCREEN_H / 2 + sinf(a) * r1
                              }, 2.0f, Col(200, 240, 255, 150));
                          }
                      if (D.sparkT > 0.0f)
                          for (int i = 0; i < 12; i++) {
                              float a = i * PI_F / 6.0f, l = 30.0f + D.sparkT * 220.0f;
                              Vector2 c = { SCREEN_W * 0.5f, SCREEN_H * 0.45f };
                              DrawLineEx((Vector2) { c.x + cosf(a) * 20.0f, c.y + sinf(a) * 20.0f }, (Vector2) { c.x + cosf(a) * l, c.y + sinf(a) * l }, 4.0f, Col(255, 230, 120, 255));
                          }
                  
                      if (D.weapon == 0) DrawSwordViewmodel(); else DrawGunViewmodel();
                  
                      float cx = SCREEN_W * 0.5f, cy = SCREEN_H * 0.5f;
                      if (D.weapon == 1) {
                          DrawLine((int)cx - 14, (int)cy, (int)cx - 5, (int)cy, RAYWHITE);
                          DrawLine((int)cx + 5, (int)cy, (int)cx + 14, (int)cy, RAYWHITE);
                          DrawLine((int)cx, (int)cy - 14, (int)cx, (int)cy - 5, RAYWHITE);
                          DrawLine((int)cx, (int)cy + 5, (int)cx, (int)cy + 14, RAYWHITE);
                      }
                      DrawCircleV((Vector2) { cx, cy }, 2.5f, RAYWHITE);
                  
                      /* telegraph marker over the enemy */
                      {
                          Vector3 hp = V3(D.ex, D.ey + 2.6f, D.ez);
                          Vector3 f = DuelForward3();
                          float dot = (hp.x - gCam.position.x) * f.x + (hp.y - gCam.position.y) * f.y + (hp.z - gCam.position.z) * f.z;
                          if (dot > 0.0f && D.phase == 1) {
                              Vector2 s = GetWorldToScreen(hp, gCam);
                              if (D.estun > 0.0f) DrawTextC("STUNNED", (int)s.x, (int)s.y - 10, 22, Col(130, 180, 255, 255));
                              else if (D.es == E_WINDUP_SWORD) DrawTextC("!", (int)s.x, (int)s.y - 30, 56, Col(255, 60, 40, 255));
                              else if (D.es == E_WINDUP_SHOT) DrawTextC("!", (int)s.x, (int)s.y - 30, 56, Col(255, 170, 40, 255));
                              else if (D.es == E_WINDUP_SLAM) DrawTextC("SLAM!", (int)s.x, (int)s.y - 20, 30, Col(255, 80, 40, 255));
                          }
                      }
                      /* floating damage numbers */
                      for (int i = 0; i < 24; i++) {
                          if (!DN[i].on) continue;
                          float f = 1.0f - DN[i].life / 0.9f;
                          Vector3 wp = V3(DN[i].p.x, DN[i].p.y + f * 1.2f, DN[i].p.z);
                          Vector3 fw = DuelForward3();
                          float dot = (wp.x - gCam.position.x) * fw.x + (wp.y - gCam.position.y) * fw.y + (wp.z - gCam.position.z) * fw.z;
                          if (dot <= 0.0f) continue;
                          Vector2 s = GetWorldToScreen(wp, gCam);
                          int fs = DN[i].val >= 20.0f ? 38 : 26;
                          Color c = DN[i].c;
                          c.a = (unsigned char)(255 * ClampF(DN[i].life / 0.4f, 0.0f, 1.0f));
                          DrawTextC(TextFormat("%d", (int)DN[i].val), (int)s.x + 2, (int)s.y + 2, fs, Col(0, 0, 0, c.a));
                          DrawTextC(TextFormat("%d", (int)DN[i].val), (int)s.x, (int)s.y, fs, c);
                      }
                  
                      /* enemy bar */
                      {
                          const char* en = TextFormat("%s %s  (Lv %d)%s", D.ef.side > 0 ? "White" : "Black", PNAME[D.ef.piece], G.aiLevel, D.playerIsAttacker ? "" : "  [ATTACKER]");
                          DrawPieceGlyph(D.ef.side * D.ef.piece, SCREEN_W / 2 - 220.0f, 36.0f, 18.0f, 1.0f);
                          DrawText(en, SCREEN_W / 2 - 192, 14, 20, RAYWHITE);
                          DrawBar(SCREEN_W / 2 - 192, 40, 384, 16, D.ehp / D.emax, Col(215, 62, 62, 255));
                      }
                      DrawText(TextFormat("TIME %02d", (int)ceilf(D.timeLeft)), SCREEN_W - 130, 16, 26, D.timeLeft < 15.0f ? Col(255, 90, 80, 255) : RAYWHITE);
                      DrawText(D.playerIsAttacker ? "You: ATTACKER (bonus stats)" : "You: DEFENDER", 20, 16, 20, D.playerIsAttacker ? GOLD : LIGHTGRAY);
                      DrawMinimap();
                  
                      /* player bar */
                      DrawPieceGlyph(D.pf.side * D.pf.piece, 40.0f, SCREEN_H - 76.0f, 18.0f, 1.0f);
                      DrawText(TextFormat("YOU - %s %s", D.pf.side > 0 ? "White" : "Black", PNAME[D.pf.piece]), 66, SCREEN_H - 92, 20, RAYWHITE);
                      DrawBar(66, SCREEN_H - 66, 360, 18, D.php / D.pmax, Col(90, 205, 95, 255));
                      DrawText(TextFormat("%d / %d", (int)ceilf(D.php), (int)D.pmax), 70, SCREEN_H - 40, 18, LIGHTGRAY);
                  
                      /* abilities */
                      float ax = SCREEN_W / 2 - 135.0f, ay = SCREEN_H - 100.0f;
                      DrawAbility(ax, ay, "Q", "Dash", D.dashCdT, D.pf.dashCd, Col(120, 220, 255, 255));
                      DrawAbility(ax + 90.0f, ay, "E", "Shock", D.shockCdT, 8.0f, Col(120, 220, 255, 255));
                      {
                          Rectangle r = { ax + 180.0f, ay, 74.0f, 74.0f };
                          DrawRectangleRounded(r, 0.18f, 6, Col(20, 22, 32, 225));
                          float f = D.ultT > 0.0f ? D.ultT / 7.0f : D.ult / 100.0f;
                          DrawRectangle((int)r.x + 3, (int)(r.y + 3 + 68.0f * (1.0f - f)), 68, (int)(68.0f * f), D.ultT > 0.0f ? Col(255, 190, 60, 210) : Col(160, 110, 40, 200));
                          if (D.ult >= 100.0f && D.ultT <= 0.0f) DrawRectangleLinesEx(r, 3.0f, Col(255, 200, 60, 255));
                          DrawText("F", (int)r.x + 7, (int)r.y + 5, 20, Col(255, 200, 60, 255));
                          DrawTextC(D.ultT > 0.0f ? "ACTIVE" : "Overdrive", (int)r.x + 37, (int)r.y + 54, 14, LIGHTGRAY);
                      }
                      if (D.weapon == 0) DrawText("[1] SWORD", SCREEN_W - 200, 60, 22, RAYWHITE);
                      else DrawText(D.reloadT > 0.0f ? "[2] GUN  reloading..." : TextFormat("[2] GUN  %d / 12", D.ammo), SCREEN_W - 260, 60, 22, RAYWHITE);
                      if (D.guarding) DrawText("GUARD", SCREEN_W - 200, 90, 22, Col(120, 220, 255, 255));
                  
                      if (D.msgT > 0.0f)
                          DrawTextC(D.msg, SCREEN_W / 2, 150, 46, Col(255, 220, 90, (int)(255 * ClampF(D.msgT / 0.5f, 0.0f, 1.0f))));
                  }
                  
                  static void DuelFrame(float dt)
                  {
                      ClearBackground(Col(8, 10, 22, 255));
                  
                      if (IsKeyPressed(KEY_ESCAPE) && D.phase != 2) {
                          D.paused = !D.paused;
                          if (D.paused) EnableCursor(); else { DisableCursor(); D.skipLook = 3; }
                      }
                      if (!D.paused) {
                          float sdt = dt;
                          D.phaseT += dt;
                          if (D.hitstop > 0.0f) { D.hitstop -= dt; sdt = 0.0f; }
                          else if (D.slowT > 0.0f) { D.slowT -= dt; sdt = dt * 0.3f; }
                          if (sdt > 0.0f) { if (UpdateDuel(sdt)) return; }
                          else PlayerLook();
                      }
                  
                      UpdateCam();
                      DrawDuelWorld();
                      DrawDuelHUD();
                  
                      if (D.phase == 0) {
                          float a = ClampF((2.8f - D.phaseT) / 0.4f, 0.0f, 1.0f);
                          DrawRectangle(0, 0, SCREEN_W, SCREEN_H, Col(0, 0, 0, (int)(165 * a)));
                          DrawTextC("DUEL!", SCREEN_W / 2 + 4, 64, 110, Col(0, 0, 0, (int)(180 * a)));
                          DrawTextC("DUEL!", SCREEN_W / 2, 60, 110, Col(255, 90, 70, (int)(255 * a)));
                          int att = G.hist[G.nply].b[D.move.from];
                          int defp = G.hist[G.nply].b[D.move.to];
                          if (defp == 0) defp = G.hist[G.nply].b[RowOf(D.move.from) * 8 + ColOf(D.move.to)];
                          DrawPieceGlyph(att, SCREEN_W / 2 - 240.0f, 270.0f, 56.0f, a);
                          DrawPieceGlyph(defp, SCREEN_W / 2 + 240.0f, 270.0f, 56.0f, a);
                          DrawTextC("VS", SCREEN_W / 2, 238, 56, Fade(RAYWHITE, a));
                          DrawTextC(TextFormat("%s  attacks  %s", PNAME[abs(att)], PNAME[abs(defp)]), SCREEN_W / 2, 330, 26, Fade(LIGHTGRAY, a));
                          DrawTextC("ATTACKER:  HP x1.7   DAMAGE x1.4   SPEED +12%   DASH CD -35%", SCREEN_W / 2, 372, 22, Fade(GOLD, a));
                          DrawTextC("DEFENDER:  HP x0.85", SCREEN_W / 2, 402, 22, Fade(Col(150, 190, 255, 255), a));
                          DrawTextC(D.playerIsAttacker ? "You are the ATTACKER" : "You are the DEFENDER - survive!", SCREEN_W / 2, 440, 28,
                              Fade(D.playerIsAttacker ? GOLD : Col(150, 190, 255, 255), a));
                          int cnt = (int)ceilf(2.8f - D.phaseT);
                          if (cnt > 3) cnt = 3;
                          if (cnt >= 1) DrawTextC(TextFormat("%d", cnt), SCREEN_W / 2, 480, 110, Fade(GOLD, a));
                          DrawTextC("WASD move | Mouse look | LMB attack | RMB guard (tap right before a hit = PARRY)", SCREEN_W / 2, 630, 22, Fade(RAYWHITE, a));
                          DrawTextC("Q dash | E shockwave | F overdrive | 1/2/Tab/Wheel weapon | R reload | ESC pause", SCREEN_W / 2, 660, 22, Fade(RAYWHITE, a));
                      }
                      else if (D.phase == 1 && D.phaseT < 0.8f) {
                          DrawTextC("FIGHT!", SCREEN_W / 2, 200, 90, Col(255, 220, 90, (int)(255 * (1.0f - D.phaseT / 0.8f))));
                      }
                  
                      if (D.phase == 2) {
                          float a = ClampF(D.phaseT / 0.6f, 0.0f, 1.0f);
                          DrawRectangle(0, 0, SCREEN_W, SCREEN_H, Col(0, 0, 0, (int)(150 * a)));
                          DrawTextC(D.playerWon ? "VICTORY" : "DEFEATED", SCREEN_W / 2 + 4, 204, 120, Col(0, 0, 0, (int)(200 * a)));
                          DrawTextC(D.playerWon ? "VICTORY" : "DEFEATED", SCREEN_W / 2, 200, 120,
                              D.playerWon ? Col(255, 220, 90, (int)(255 * a)) : Col(230, 70, 60, (int)(255 * a)));
                          DrawTextC(D.attackerWins ? "The capture succeeds!" : "The attacker is slain - the defender holds the square!",
                              SCREEN_W / 2, 360, 30, Fade(RAYWHITE, a));
                          if (D.phaseT > 1.3f)
                              DrawTextC("Click or press ENTER to return to the board", SCREEN_W / 2, 470, 24, Fade(LIGHTGRAY, 0.5f + 0.5f * sinf((float)GetTime() * 4.0f)));
                      }
                  
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
                      DrawRectangleGradientV(0, 0, SCREEN_W, SCREEN_H, (Color) { 38, 40, 58, 255 }, (Color) { 16, 17, 26, 255 });
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
                  
                      G.mode = 1;
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
