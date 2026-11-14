20261022 이동현 심재창 교수님 과제
c언어와 raylib를 이용해서 체스 게임을 만들었습니다.

```
#include "raylib.h"
#include <stdlib.h>
#include <string.h>

#define CELL 80
#define BOARD 640
#define PANEL 240
#define WIN_W (BOARD + PANEL)
#define WIN_H BOARD
#define MATE 100000
#define INF 1000000

enum { EMPTY, PAWN, KNIGHT, BISHOP, ROOK, QUEEN, KING };
enum { WK_C = 1, WQ_C = 2, BK_C = 4, BQ_C = 8 };

typedef struct {
    signed char b[64];
    int side, castle, ep, half;
} Pos;

typedef struct {
    unsigned char from, to, promo;
    int score;
} Move;

static const int VAL[7] = {0, 100, 320, 330, 500, 900, 0};
static const int DR[8] = {-1, -1, -1, 0, 0, 1, 1, 1};
static const int DC[8] = {-1, 0, 1, -1, 1, -1, 0, 1};
static const int KR[8] = {-2, -2, -1, -1, 1, 1, 2, 2};
static const int KC[8] = {-1, 1, -2, 2, -2, 2, -1, 1};

static const int PST[7][64] = {
    {0},
    {0, 0, 0, 0, 0, 0, 0, 0,
     50, 50, 50, 50, 50, 50, 50, 50,
     10, 10, 20, 30, 30, 20, 10, 10,
     5, 5, 10, 25, 25, 10, 5, 5,
     0, 0, 0, 20, 20, 0, 0, 0,
     5, -5, -10, 0, 0, -10, -5, 5,
     5, 10, 10, -20, -20, 10, 10, 5,
     0, 0, 0, 0, 0, 0, 0, 0},
    {-50, -40, -30, -30, -30, -30, -40, -50,
     -40, -20, 0, 0, 0, 0, -20, -40,
     -30, 0, 10, 15, 15, 10, 0, -30,
     -30, 5, 15, 20, 20, 15, 5, -30,
     -30, 0, 15, 20, 20, 15, 0, -30,
     -30, 5, 10, 15, 15, 10, 5, -30,
     -40, -20, 0, 5, 5, 0, -20, -40,
     -50, -40, -30, -30, -30, -30, -40, -50},
    {-20, -10, -10, -10, -10, -10, -10, -20,
     -10, 0, 0, 0, 0, 0, 0, -10,
     -10, 0, 5, 10, 10, 5, 0, -10,
     -10, 5, 5, 10, 10, 5, 5, -10,
     -10, 0, 10, 10, 10, 10, 0, -10,
     -10, 10, 10, 10, 10, 10, 10, -10,
     -10, 5, 0, 0, 0, 0, 5, -10,
     -20, -10, -10, -10, -10, -10, -10, -20},
    {0, 0, 0, 0, 0, 0, 0, 0,
     5, 10, 10, 10, 10, 10, 10, 5,
     -5, 0, 0, 0, 0, 0, 0, -5,
     -5, 0, 0, 0, 0, 0, 0, -5,
     -5, 0, 0, 0, 0, 0, 0, -5,
     -5, 0, 0, 0, 0, 0, 0, -5,
     -5, 0, 0, 0, 0, 0, 0, -5,
     0, 0, 0, 5, 5, 0, 0, 0},
    {-20, -10, -10, -5, -5, -10, -10, -20,
     -10, 0, 0, 0, 0, 0, 0, -10,
     -10, 0, 5, 5, 5, 5, 0, -10,
     -5, 0, 5, 5, 5, 5, 0, -5,
     0, 0, 5, 5, 5, 5, 0, -5,
     -10, 5, 5, 5, 5, 5, 0, -10,
     -10, 0, 5, 0, 0, 0, 0, -10,
     -20, -10, -10, -5, -5, -10, -10, -20},
    {-30, -40, -40, -50, -50, -40, -40, -30,
     -30, -40, -40, -50, -50, -40, -40, -30,
     -30, -40, -40, -50, -50, -40, -40, -30,
     -30, -40, -40, -50, -50, -40, -40, -30,
     -20, -30, -30, -40, -40, -30, -30, -20,
     -10, -20, -20, -20, -20, -20, -20, -10,
     20, 20, 0, 0, 0, 0, 20, 20,
     20, 30, 10, 0, 0, 10, 30, 20}};

static const int KING_END[64] = {
    -50, -40, -30, -20, -20, -30, -40, -50,
    -30, -20, -10, 0, 0, -10, -20, -30,
    -30, -10, 20, 30, 30, 20, -10, -30,
    -30, -10, 30, 40, 40, 30, -10, -30,
    -30, -10, 30, 40, 40, 30, -10, -30,
    -30, -10, 20, 30, 30, 20, -10, -30,
    -30, -30, 0, 0, 0, 0, -30, -30,
    -50, -30, -30, -30, -30, -30, -30, -50};

static int attacked(const Pos *p, int sq, int by) {
    int r = sq >> 3, c = sq & 7;
    int pr = r + (by == 1 ? 1 : -1);
    if (pr >= 0 && pr < 8) {
        if (c > 0 && p->b[pr * 8 + c - 1] == by * PAWN) return 1;
        if (c < 7 && p->b[pr * 8 + c + 1] == by * PAWN) return 1;
    }
    for (int i = 0; i < 8; i++) {
        int nr = r + KR[i], nc = c + KC[i];
        if (nr >= 0 && nr < 8 && nc >= 0 && nc < 8 && p->b[nr * 8 + nc] == by * KNIGHT) return 1;
    }
    for (int d = 0; d < 8; d++) {
        int nr = r + DR[d], nc = c + DC[d];
        if (nr >= 0 && nr < 8 && nc >= 0 && nc < 8 && p->b[nr * 8 + nc] == by * KING) return 1;
        int diag = DR[d] && DC[d];
        nr = r + DR[d];
        nc = c + DC[d];
        while (nr >= 0 && nr < 8 && nc >= 0 && nc < 8) {
            int pc = p->b[nr * 8 + nc];
            if (pc) {
                if (pc * by > 0) {
                    int t = abs(pc);
                    if (t == QUEEN || (diag && t == BISHOP) || (!diag && t == ROOK)) return 1;
                }
                break;
            }
            nr += DR[d];
            nc += DC[d];
        }
    }
    return 0;
}

static int find_king(const Pos *p, int side) {
    for (int i = 0; i < 64; i++)
        if (p->b[i] == side * KING) return i;
    return -1;
}

static int add(Move *m, int n, int f, int t, int promo) {
    m[n].from = f;
    m[n].to = t;
    m[n].promo = promo;
    m[n].score = 0;
    return n + 1;
}

static int gen_pseudo(const Pos *p, Move *m, int caps) {
    int n = 0, s = p->side;
    for (int sq = 0; sq < 64; sq++) {
        int pc = p->b[sq];
        if (pc * s <= 0) continue;
        int t = abs(pc), r = sq >> 3, c = sq & 7;
        if (t == PAWN) {
            int dir = -s, nr = r + dir;
            if (nr < 0 || nr > 7) continue;
            int last = (nr == 0 || nr == 7), f = nr * 8 + c;
            if (!p->b[f]) {
                if (last) {
                    for (int pr = QUEEN; pr >= KNIGHT; pr--) n = add(m, n, sq, f, pr);
                } else if (!caps) {
                    n = add(m, n, sq, f, 0);
                    int start = (s == 1) ? 6 : 1;
                    if (r == start && !p->b[f + dir * 8]) n = add(m, n, sq, f + dir * 8, 0);
                }
            }
            for (int dc = -1; dc <= 1; dc += 2) {
                int nc = c + dc;
                if (nc < 0 || nc > 7) continue;
                int to = nr * 8 + nc;
                if (p->b[to] * s < 0 || (to == p->ep && !p->b[to])) {
                    if (last) {
                        for (int pr = QUEEN; pr >= KNIGHT; pr--) n = add(m, n, sq, to, pr);
                    } else {
                        n = add(m, n, sq, to, 0);
                    }
                }
            }
        } else if (t == KNIGHT || t == KING) {
            for (int i = 0; i < 8; i++) {
                int dr = (t == KNIGHT) ? KR[i] : DR[i];
                int dc = (t == KNIGHT) ? KC[i] : DC[i];
                int nr = r + dr, nc = c + dc;
                if (nr < 0 || nr > 7 || nc < 0 || nc > 7) continue;
                int tgt = p->b[nr * 8 + nc];
                if (tgt * s > 0) continue;
                if (caps && !tgt) continue;
                n = add(m, n, sq, nr * 8 + nc, 0);
            }
            if (t == KING && !caps) {
                if (s == 1 && sq == 60) {
                    if ((p->castle & WK_C) && !p->b[61] && !p->b[62] && p->b[63] == ROOK &&
                        !attacked(p, 60, -1) && !attacked(p, 61, -1))
                        n = add(m, n, 60, 62, 0);
                    if ((p->castle & WQ_C) && !p->b[59] && !p->b[58] && !p->b[57] && p->b[56] == ROOK &&
                        !attacked(p, 60, -1) && !attacked(p, 59, -1))
                        n = add(m, n, 60, 58, 0);
                } else if (s == -1 && sq == 4) {
                    if ((p->castle & BK_C) && !p->b[5] && !p->b[6] && p->b[7] == -ROOK &&
                        !attacked(p, 4, 1) && !attacked(p, 5, 1))
                        n = add(m, n, 4, 6, 0);
                    if ((p->castle & BQ_C) && !p->b[3] && !p->b[2] && !p->b[1] && p->b[0] == -ROOK &&
                        !attacked(p, 4, 1) && !attacked(p, 3, 1))
                        n = add(m, n, 4, 2, 0);
                }
            }
        } else {
            for (int d = 0; d < 8; d++) {
                int diag = DR[d] && DC[d];
                if (t == BISHOP && !diag) continue;
                if (t == ROOK && diag) continue;
                int nr = r + DR[d], nc = c + DC[d];
                while (nr >= 0 && nr < 8 && nc >= 0 && nc < 8) {
                    int tgt = p->b[nr * 8 + nc];
                    if (tgt * s > 0) break;
                    if (!(caps && !tgt)) n = add(m, n, sq, nr * 8 + nc, 0);
                    if (tgt) break;
                    nr += DR[d];
                    nc += DC[d];
                }
            }
        }
    }
    return n;
}

static int castle_mask(int sq) {
    switch (sq) {
    case 0: return BQ_C;
    case 7: return BK_C;
    case 56: return WQ_C;
    case 63: return WK_C;
    case 4: return BK_C | BQ_C;
    case 60: return WK_C | WQ_C;
    }
    return 0;
}

static Pos apply(const Pos *src, Move m) {
    Pos p = *src;
    int pc = p.b[m.from], s = p.side, t = abs(pc);
    int cap = p.b[m.to] != 0;
    p.half++;
    if (t == PAWN || cap) p.half = 0;
    if (t == PAWN && m.to == p.ep && !cap && (m.from & 7) != (m.to & 7))
        p.b[(m.from >> 3) * 8 + (m.to & 7)] = 0;
    p.ep = -1;
    if (t == PAWN && abs((int)m.to - (int)m.from) == 16) p.ep = (m.from + m.to) / 2;
    p.b[m.to] = pc;
    p.b[m.from] = 0;
    if (t == KING && abs((int)m.to - (int)m.from) == 2) {
        if (m.to > m.from) {
            p.b[m.to - 1] = p.b[m.to + 1];
            p.b[m.to + 1] = 0;
        } else {
            p.b[m.to + 1] = p.b[m.to - 2];
            p.b[m.to - 2] = 0;
        }
    }
    if (t == PAWN && (m.to >> 3) == (s == 1 ? 0 : 7)) p.b[m.to] = s * (m.promo ? m.promo : QUEEN);
    p.castle &= ~(castle_mask(m.from) | castle_mask(m.to));
    p.side = -s;
    return p;
}

static int gen_legal(const Pos *p, Move *out, int caps) {
    Move tmp[256];
    int n = gen_pseudo(p, tmp, caps), k = 0;
    for (int i = 0; i < n; i++) {
        Pos q = apply(p, tmp[i]);
        int ks = find_king(&q, p->side);
        if (ks >= 0 && !attacked(&q, ks, -p->side)) out[k++] = tmp[i];
    }
    return k;
}

static int evaluate(const Pos *p) {
    int score = 0, npm = 0;
    for (int i = 0; i < 64; i++) {
        int t = abs(p->b[i]);
        if (t != EMPTY && t != PAWN && t != KING) npm += VAL[t];
    }
    int endgame = npm <= 2600;
    for (int i = 0; i < 64; i++) {
        int pc = p->b[i];
        if (!pc) continue;
        int t = abs(pc), idx = pc > 0 ? i : (i ^ 56);
        int ps = (t == KING && endgame) ? KING_END[idx] : PST[t][idx];
        int v = VAL[t] + ps;
        score += pc > 0 ? v : -v;
    }
    return score * p->side;
}

static int cmp_move(const void *a, const void *b) {
    return ((const Move *)b)->score - ((const Move *)a)->score;
}

static void order(const Pos *p, Move *m, int n) {
    for (int i = 0; i < n; i++) {
        int victim = abs(p->b[m[i].to]), att = abs(p->b[m[i].from]);
        if (!victim && att == PAWN && (m[i].to & 7) != (m[i].from & 7)) victim = PAWN;
        m[i].score = victim ? 10 * victim - att + 100 : 0;
        if (m[i].promo) m[i].score += m[i].promo * 50;
    }
    qsort(m, n, sizeof(Move), cmp_move);
}

static int quiesce(const Pos *p, int alpha, int beta) {
    int stand = evaluate(p);
    if (stand >= beta) return beta;
    if (stand > alpha) alpha = stand;
    Move ml[256];
    int n = gen_legal(p, ml, 1);
    order(p, ml, n);
    for (int i = 0; i < n; i++) {
        Pos q = apply(p, ml[i]);
        int sc = -quiesce(&q, -beta, -alpha);
        if (sc >= beta) return beta;
        if (sc > alpha) alpha = sc;
    }
    return alpha;
}

static int search(const Pos *p, int depth, int alpha, int beta, int ply) {
    if (p->half >= 100) return 0;
    int ks = find_king(p, p->side);
    int chk = ks >= 0 && attacked(p, ks, -p->side);
    if (chk && ply < 10) depth++;
    if (depth <= 0) return quiesce(p, alpha, beta);
    Move ml[256];
    int n = gen_legal(p, ml, 0);
    if (n == 0) return chk ? -MATE + ply : 0;
    order(p, ml, n);
    for (int i = 0; i < n; i++) {
        Pos q = apply(p, ml[i]);
        int sc = -search(&q, depth - 1, -beta, -alpha, ply + 1);
        if (sc >= beta) return beta;
        if (sc > alpha) alpha = sc;
    }
    return alpha;
}

static Move best_move(const Pos *p, int depth) {
    Move ml[256];
    int n = gen_legal(p, ml, 0);
    order(p, ml, n);
    int alpha = -INF, best = 0;
    for (int i = 0; i < n; i++) {
        Pos q = apply(p, ml[i]);
        int sc = -search(&q, depth - 1, -INF, -alpha, 1);
        if (sc > alpha || i == 0) {
            alpha = sc;
            best = i;
        }
    }
    return ml[best];
}

static Pos pos;
static Move legal[256];
static int nlegal, human = 1, flip = 0, depth = 3, sel = -1;
static int lastFrom = -1, lastTo = -1, promoPending = 0, pFrom, pTo;
static int over = 0, think = 0, inCheck = 0;
static const char *msg = "";

static int insufficient(void) {
    int minors = 0;
    for (int i = 0; i < 64; i++) {
        int t = abs(pos.b[i]);
        if (t == PAWN || t == ROOK || t == QUEEN) return 0;
        if (t == KNIGHT || t == BISHOP) minors++;
    }
    return minors <= 1;
}

static void update_status(void) {
    nlegal = gen_legal(&pos, legal, 0);
    int ks = find_king(&pos, pos.side);
    inCheck = attacked(&pos, ks, -pos.side);
    over = 0;
    msg = "";
    if (!nlegal) {
        over = 1;
        msg = inCheck ? (pos.side == 1 ? "Checkmate! Black wins" : "Checkmate! White wins") : "Stalemate - draw";
    } else if (pos.half >= 100) {
        over = 1;
        msg = "Draw - 50 move rule";
    } else if (insufficient()) {
        over = 1;
        msg = "Draw - insufficient material";
    }
}

static void new_game(int side) {
    static const int back[8] = {ROOK, KNIGHT, BISHOP, QUEEN, KING, BISHOP, KNIGHT, ROOK};
    memset(&pos, 0, sizeof pos);
    for (int c = 0; c < 8; c++) {
        pos.b[c] = -back[c];
        pos.b[8 + c] = -PAWN;
        pos.b[48 + c] = PAWN;
        pos.b[56 + c] = back[c];
    }
    pos.side = 1;
    pos.castle = 15;
    pos.ep = -1;
    human = side;
    flip = (side == -1);
    sel = -1;
    lastFrom = lastTo = -1;
    promoPending = 0;
    think = 0;
    update_status();
}

static void do_move(Move m) {
    pos = apply(&pos, m);
    lastFrom = m.from;
    lastTo = m.to;
    sel = -1;
    promoPending = 0;
    think = 0;
    update_status();
}

static void sq_xy(int sq, int *x, int *y) {
    int r = sq >> 3, c = sq & 7;
    if (flip) {
        r = 7 - r;
        c = 7 - c;
    }
    *x = c * CELL;
    *y = r * CELL;
}

static void draw_piece(int pc, int x, int y) {
    int t = abs(pc);
    Color fill = pc > 0 ? (Color){250, 250, 250, 255} : (Color){35, 35, 35, 255};
    Color edge = pc > 0 ? (Color){30, 30, 30, 255} : (Color){230, 230, 230, 255};
    Color tx = pc > 0 ? BLACK : WHITE;
    int cx = x + CELL / 2, cy = y + CELL / 2;
    DrawCircle(cx, cy, 32, edge);
    DrawCircle(cx, cy, 29, fill);
    char s[2] = {"?PNBRQK"[t], 0};
    int w = MeasureText(s, 40);
    DrawText(s, cx - w / 2, cy - 20, 40, tx);
}

static const int PROMOS[4] = {QUEEN, ROOK, BISHOP, KNIGHT};

static void draw_board(void) {
    int ksq = find_king(&pos, pos.side);
    for (int sq = 0; sq < 64; sq++) {
        int x, y;
        sq_xy(sq, &x, &y);
        int light = (((sq >> 3) + (sq & 7)) & 1) == 0;
        DrawRectangle(x, y, CELL, CELL, light ? (Color){240, 217, 181, 255} : (Color){181, 136, 99, 255});
        if (sq == lastFrom || sq == lastTo) DrawRectangle(x, y, CELL, CELL, (Color){255, 235, 60, 90});
        if (inCheck && sq == ksq) DrawRectangle(x, y, CELL, CELL, (Color){230, 40, 40, 140});
        if (sq == sel) DrawRectangle(x, y, CELL, CELL, (Color){60, 200, 90, 120});
    }
    for (int i = 0; i < 8; i++) {
        char f[2] = {(char)(flip ? 'h' - i : 'a' + i), 0};
        char rk[2] = {(char)(flip ? '1' + i : '8' - i), 0};
        DrawText(f, i * CELL + CELL - 12, BOARD - 16, 12, (Color){40, 30, 20, 220});
        DrawText(rk, 4, i * CELL + 4, 12, (Color){40, 30, 20, 220});
    }
    for (int sq = 0; sq < 64; sq++) {
        if (!pos.b[sq]) continue;
        int x, y;
        sq_xy(sq, &x, &y);
        draw_piece(pos.b[sq], x, y);
    }
    if (sel >= 0 && !over && pos.side == human) {
        for (int i = 0; i < nlegal; i++) {
            if (legal[i].from != sel) continue;
            if (legal[i].promo && legal[i].promo != QUEEN) continue;
            int to = legal[i].to, x, y;
            sq_xy(to, &x, &y);
            int cap = pos.b[to] || (abs(pos.b[sel]) == PAWN && (to & 7) != (sel & 7));
            if (cap)
                DrawRing((Vector2){x + CELL / 2, y + CELL / 2}, 32, 38, 0, 360, 32, (Color){40, 160, 70, 200});
            else
                DrawCircle(x + CELL / 2, y + CELL / 2, 12, (Color){40, 160, 70, 180});
        }
    }
    if (promoPending) {
        DrawRectangle(0, 0, BOARD, BOARD, (Color){0, 0, 0, 140});
        int bx = BOARD / 2 - 160, by = BOARD / 2 - 40;
        DrawRectangle(bx - 6, by - 6, 332, 92, (Color){60, 60, 66, 255});
        for (int i = 0; i < 4; i++) {
            DrawRectangle(bx + i * CELL, by, CELL, CELL, (Color){200, 200, 205, 255});
            draw_piece(PROMOS[i] * human, bx + i * CELL, by);
        }
    }
}

static void draw_panel(void) {
    DrawRectangle(BOARD, 0, PANEL, WIN_H, (Color){30, 30, 34, 255});
    int px = BOARD + 16;
    DrawText("CHESS vs AI", px, 20, 26, RAYWHITE);
    DrawText(human == 1 ? "You: White" : "You: Black", px, 70, 20, LIGHTGRAY);
    DrawText(TextFormat("AI level: %d", depth), px, 98, 20, LIGHTGRAY);
    if (over) {
        DrawText(msg, px, 150, 18, GOLD);
    } else {
        DrawText(pos.side == human ? "Your turn" : "AI thinking...", px, 150, 20, RAYWHITE);
        if (inCheck) DrawText("Check!", px, 180, 20, RED);
    }
    DrawText("R   restart", px, 400, 18, GRAY);
    DrawText("C   switch side", px, 428, 18, GRAY);
    DrawText("1-4 AI level", px, 456, 18, GRAY);
    DrawText("ESC quit", px, 484, 18, GRAY);
    DrawText("Click piece, then target", px, 540, 16, GRAY);
}

static void handle_input(void) {
    if (IsKeyPressed(KEY_R)) new_game(human);
    if (IsKeyPressed(KEY_C)) new_game(-human);
    for (int k = KEY_ONE; k <= KEY_FOUR; k++)
        if (IsKeyPressed(k)) depth = k - KEY_ONE + 1;
    if (over || pos.side != human) return;
    if (!IsMouseButtonPressed(MOUSE_BUTTON_LEFT)) return;
    Vector2 mp = GetMousePosition();
    if (promoPending) {
        int bx = BOARD / 2 - 160, by = BOARD / 2 - 40;
        for (int i = 0; i < 4; i++) {
            if (mp.x >= bx + i * CELL && mp.x < bx + (i + 1) * CELL && mp.y >= by && mp.y < by + CELL) {
                for (int j = 0; j < nlegal; j++)
                    if (legal[j].from == pFrom && legal[j].to == pTo && legal[j].promo == PROMOS[i]) {
                        do_move(legal[j]);
                        return;
                    }
            }
        }
        promoPending = 0;
        sel = -1;
        return;
    }
    if (mp.x < 0 || mp.x >= BOARD || mp.y < 0 || mp.y >= BOARD) return;
    int c = (int)mp.x / CELL, r = (int)mp.y / CELL;
    if (flip) {
        c = 7 - c;
        r = 7 - r;
    }
    int sq = r * 8 + c;
    if (sel >= 0) {
        for (int i = 0; i < nlegal; i++) {
            if (legal[i].from == sel && legal[i].to == sq) {
                if (legal[i].promo) {
                    promoPending = 1;
                    pFrom = sel;
                    pTo = sq;
                } else {
                    do_move(legal[i]);
                }
                return;
            }
        }
    }
    sel = (pos.b[sq] * human > 0) ? sq : -1;
}

static void ai_step(void) {
    if (over || promoPending || pos.side == human) return;
    if (think < 2) {
        think++;
    } else {
        do_move(best_move(&pos, depth));
    }
}

int main(void) {
    InitWindow(WIN_W, WIN_H, "Chess");
    SetTargetFPS(60);
    new_game(1);
    while (!WindowShouldClose()) {
        handle_input();
        ai_step();
        BeginDrawing();
        ClearBackground((Color){30, 30, 34, 255});
        draw_board();
        draw_panel();
        EndDrawing();
    }
    CloseWindow();
    return 0;
}
```
