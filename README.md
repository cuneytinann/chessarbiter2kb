**[English](#chessarbiter2kb)** · **[Türkçe](#turkce)**

# chessarbiter2kb

A two-player chess arbiter in **1,999 bytes** of one HTML file, and its twin in **1,909 bytes**. No libraries, no build step, no server, no packer. Download a file, double-click, play.

Both files enforce the same rules. `index.html` stores the board as letters, the way a FEN does. `hexadecimal.html` stores it as numbers. Put side by side, they are a small course in how few characters the rules of chess need, and in what a choice of representation costs.

Part of the [Golfstack](https://www.fidelite.art/) project.

## Play

| file | board | GitHub Pages | project site |
| --- | --- | --- | --- |
| `index.html` | letters | [chessarbiter2kb](https://cuneytinann.github.io/chessarbiter2kb/) | [L2_string_flip_noBlockedDetector.html](https://www.fidelite.art/special/outofLevels/L2_string_flip_noBlockedDetector.html) |
| `hexadecimal.html` | numbers | [hexadecimal.html](https://cuneytinann.github.io/chessarbiter2kb/hexadecimal.html) | [L2_noBlockedDetector.html](https://www.fidelite.art/special/outofLevels/L2_noBlockedDetector.html) |

On the project site both builds sit under `special/outofLevels`, as variants of the `L2` rules level.

Click a piece, click a destination. Legal targets turn amber, the selected square and then the last destination keep a green outline, and the board flips to the side to move after every ply.

## Read it in the browser

Nothing here is packed or minified by a tool. **Right-click → View Page Source** (`Ctrl` `U`, or `⌥` `⌘` `U` on macOS) shows the entire program, and that is the intended way to use this repository: open the page, play a few moves, then read the source that just refereed them.

The script is one comma-separated declaration chain in dependency order, broken so that one line holds one idea. The two files are laid out line for line in parallel (56 and 54 lines), so the same rule sits at the same height in both. Line breaks carry no meaning: strip them from either script and the syntax tree is identical.

```
aliases → state → G → V → L → C → M → setup → d → A → S
          geometry · attack · legality · castling rights · make move · draw · apply and judge · click
```

---

## What's in it

- **All piece movement**, derived from arithmetic. No direction tables, no offset arrays.
- **Full legality.** A move that leaves your own king in check is never accepted.
- **Castling**, both sides, with every condition: right still held, rook path clear, king not in check, king not crossing an attacked square, king not landing in check.
- **En passant**, including the case almost everyone gets wrong (Lesson 3).
- **Promotion with a picker.** Queen, rook, bishop, knight. The board locks until you choose.
- **Check, checkmate, stalemate**, told apart.
- **Insufficient material** to FIDE's reading (Lesson 4).
- **5-fold repetition** and the **75-move rule**, declared without a claim.
- **Status line**: corner coordinates, `C!` when the side to move is in check, `M=` the halfmove clock, `R=` how many times the current position has occurred. When the game ends, the line becomes the result code.

## What's not in it

- No clock.
- No draw offer and no draw claim, so no 3-fold claim and no 50-move claim. The counters run and are shown; only the automatic thresholds end the game.
- No detection of **blocked positions**, the other half of FIDE Article 5.2.2, where material is sufficient but the pawns have locked and mate is impossible anyway. In FideLite's `L2` builds that detector (`J`) is 457 bytes on the numeric board and 702 on the letter board, and it was removed deliberately. The result code is therefore `IM` (insufficient material) rather than `DP` (dead position): the code names what is actually tested. A blocked position still ends as a draw, later, by repetition or by the 75-move rule.
- No bot, no undo, no FEN import or export, no PGN.

For the full arbiter — clock, draw offers and claims, resignation, flag fall, dead positions, fifteen result codes — see [fidelite.art](https://www.fidelite.art/).

## Result codes

| code | meaning |
| --- | --- |
| `W#` | White delivers mate |
| `B#` | Black delivers mate |
| `SM` | Stalemate |
| `IM` | Insufficient material |
| `5R` | 5-fold repetition |
| `75` | 75-move rule |

---

## Lesson 1 — The state is a FEN record

```js
b=[...`rnbqkbnrpppppppp${(_='-').repeat(32)}PPPPPPPPRNBQKBNR`],t=1,c=15,e=_,n=0,   // index.html
b=[...'5d37b3d599999999'+'0'.repeat(32)+'888888884c26a2c4'].map(u=>'0x'+u-0),t=1,c=15,e=-1,n=0,  // hexadecimal.html
```

Both lines are a FEN, field for field and in FEN's order: **piece placement, side to move, castling rights, en passant square, halfmove clock.** The fullmove number is the only field left out, because no rule reads it: the 50- and 75-move rules count plies, and repetition is decided by position, not by move number.

- **Letters.** Index 0 is a8, the order a FEN is written in. Uppercase is White, lowercase is Black, `-` is an empty square.
- **Numbers.** Index 0 is a1. Each square holds one hexadecimal digit: `0` empty, then `type×2 + colour`. Both files build their sixty-four characters the same way and in the same three pieces: sixteen for one side, `repeat(32)` for the empty middle, sixteen for the other. Read the two lines against each other and the same square sits at the same offset in both.

The numeric board used to write its middle as `10n**40n-10n**32n`, a BigInt evaluating to `99999999` followed by thirty-two zeros — six bytes shorter, and a full stop for anyone reading. It was spelled out on purpose. These files are meant to be read, and a board that has to be computed before it can be seen defeats the point of showing it.

The repetition key is the same four-variable concatenation in both files:

```js
$=R[s=b+t+e+c]=-~R[s]     // board, side to move, en passant square, castling rights
```

`-~undefined` is `1`, so a position seen for the first time needs no default. On the letter board the key reads almost like a FEN when printed.

## Lesson 2 — Geometry is arithmetic

`G(i,f)` answers one question: can the piece on square `i` reach square `f`, ignoring everything but the board? It derives everything from two numbers, the file distance `h` and the rank distance `v`:

```js
h=a(i%8-f%8),v=a(y-(f>>3))     file and rank distance
d=h|v                          squares spanned by a line move: h|v is max(h,v) when h==0, v==0 or h==v
k=(f-i)/d                      the step vector — ±1, ±8, ±7, ±9 — from one division
h*v==2                         the knight: only 1×2 and 2×1 give 2
T|v|h^2?d<2:…                  the king: one square, or two along the rank as castling
i+3.5*k-.5                     the castling rook: i+3 kingside, i-4 queenside, one expression
h*v==1                         a pawn capture: one file and one rank
66>>y                          the pawn home ranks: 66 is 0b1000010, only bits 1 and 6 are set
f%56<8                         the last rank, for either colour
f^8                            the en passant victim: XOR with 8 steps one rank back, in the right direction
(i/8^i)%2                      the colour of a square: rank parity XOR file parity
```

Sliding pieces walk the line one step at a time with `S`, stopping at the target or at the first occupied square. Which pieces may walk which lines is where the two files first diverge, and it is worth reading both:

```js
(h*v?h==v&P<'R':P>'B')&S()     letters: B < Q < R in the alphabet, so P<'R' lets B and Q move
                               diagonally and P>'B' lets Q and R move straight
(h*v?h==v:2)&P&&S()            numbers: bishop=1, rook=2, queen=3, so the type itself is a bit mask;
                               a diagonal yields 1, a straight line 2, and & does the rest
```

There is a second lesson hidden in the operator. On the letter board a read past the edge returns `undefined`, which is not `'-'`, so a walk on a non-line stops by itself and `&` is safe. On the numeric board `!undefined` is `true`, so the walk must be guarded with `&&`: `S` runs only after the capability test has confirmed a real line, where the step is a whole number and every square it visits is on the board.

## Lesson 3 — Legality: play it, ask, take it back

Legality is not a second move generator. It is one trick:

- `V(s)` asks the whole board whether a square is attacked — by default, the king of side `s`.
- `L` plays the candidate move on a cloned board with `M`, asks `V`, and restores the board.
- `M` writes a move and nothing else, because `L` calls it as a trial. `A` is the only function that makes anything permanent.

The dependency runs backwards in exactly two places, and both come from the rules, not from the code. `G` calls `V` because castling cannot be judged without asking whether the king's crossing square is attacked. `M` calls `L` because the en passant square may only be written if the capture is genuinely legal:

```js
e=P&d>9&&(e=q,[f-1,f+1].some(x=>b[x]=='Pp'[t]&&L(x,e)))?e:_      // letters
e=P&d>9&&(e=q,[f-1,f+1].some(x=>b[x]==17-p&&~L(x)[Q](e)))?e:-1   // numbers: 17-p is the opposing pawn
```

A pawn has just moved two squares. The naive version writes the en passant square at once. This one writes it provisionally, asks whether a neighbouring enemy pawn can *legally* capture there, and reverts if not.

The difference never shows in the capture itself — an illegal capture is rejected by the move generator either way. It shows in the **repetition counter**, because `e` is part of the position key. FIDE treats two positions as the same only if the same moves are available in both, en passant included. Record a square nobody can use, and one position lands under two keys: a repetition draw fires late, or never.

The classic case: Black plays g7–g5 while the white pawn on f5 is pinned to its king on f1 by a rook on f8. `fxg6` would expose the king, so the capture is illegal, so the square is never recorded. The check costs **48 bytes** on the letter board and **50** on the numeric one, over the naive `e=P&d>9?q:_`. It also absorbs the edge case of `[f-1,f+1]`: a neighbour index that wraps onto the other side of the board holds a pawn that `G` rejects, so the square stays unset.

## Lesson 4 — The rules everyone skips are cheap

Chess programs in code-golf collections routinely drop the halfmove clock, the repetition counter and the insufficient-material test as expensive bookkeeping. Measured in these files:

| rule | letters | bytes | numbers | bytes |
| --- | --- | --- | --- | --- |
| Halfmove clock | `n=P\|b[f]>_?0:n+1` | 16 | `n=P\|b[f]?0:n+1` | 14 |
| Repetition counter | `$=R[s=b+t+e+c]=-~R[s]` | 21 | same | 21 |
| Castling rights, all four | `C=i=>'20003001'[i%56]<<i/28` | 27 | same | 27 |
| Insufficient material, both sides | `(m=W=0,b.map((p,i)=>p>_&&(j(p)<'C'?m\|=(i/8^i)%2+1:W+=j(p)>'N'?9:j(p)=='N')),W*2+m<3)` | 84 | `(m=W=0,b.map((p,i)=>p&&(p<4?m\|=(i/8^i)%2+1:W+=p<10?9:p>11)),W*2+m<3)` | 68 |
| Every ending | `?'IM':$>4?'5R':n>149&&'75':V(t)?t?'B#':'W#':'SM'` | 48 | `?'IM':$>4?'5R':n>149?'75':0:V(t)?'WB'[t]+'#':'SM'` | 49 |
| **total** | | **196** | | **179** |

Under a tenth of either file. The expensive part of a chess program was never the rulebook.

**Castling rights** are one string: the digit under a square of the first or last rank of the array is the right that square forfeits when a piece leaves it or lands on it, shifted by `i/28` — 0 on one end of the array, 2 on the other. King and rook squares carry their bits; every other square yields `0`, and `A` clears both squares of every move with `c&=~C(i)&~C(f)`.

**Insufficient material** is one pass and one comparison. `m` collects the square colours of the bishops as bits (1 or 2; both colours make 3). `W` counts a knight as 1 and any pawn, rook or queen as 9; kings add nothing. Then `W*2+m<3` holds exactly for bare kings, a single knight, or bishops that all stand on one square colour — on either side. Two knights, bishop and knight, opposite-coloured bishops, or a knight on each side all come out at 3 or more, and play continues.

**Claims and declarations.** A draw that must be *claimed* needs a player to claim it, which needs a button. This build has no such layer, so it keeps only what an arbiter declares on its own: 5-fold repetition, the 75-move rule, insufficient material, mate and stalemate. You can still watch `R=` climb to 3 and `M=` pass 99 — under FIDE those two thresholds are claims, not declarations, so they do not end the game here.

## Lesson 5 — Letters or numbers

A letter says one thing: which piece. Colour has to be recovered from its case, and emptiness from a separate character, so almost every question asks for an extra comparison or a case conversion (`j` is `toUpperCase`). A number can be chosen so that the questions are already answered by its bits:

| code | 0 | 2 · 3 | 4 · 5 | 6 · 7 | 8 · 9 | a · b | c · d |
| --- | --- | --- | --- | --- | --- | --- | --- |
| piece | empty | bishop | rook | queen | pawn | king | knight |

Bit 0 is the colour (1 is White). The type, `p>>1`, runs bishop 1, rook 2, queen 3, pawn 4, king 5, knight 6. That order is not alphabetical, it is functional: the slider types double as direction masks (Lesson 2), and `G` can branch with plain comparisons — `P>5` knight, `P>4` king, `P>3` pawn, the rest slide.

The same questions, asked of each board:

| question | letters | bytes | numbers | bytes |
| --- | --- | --- | --- | --- |
| empty square | `b[i]==_` | 7 | `!b[i]` | 5 |
| occupied square | `b[f]>_` | 6 | `b[f]` | 4 |
| a candidate attacker of side `s` (in `V`) | `p>_&p<U!=s` | 10 | `p&1^s` | 5 |
| is it a knight | `P=='N'` | 6 | `P>5` | 3 |
| is it a pawn (in `M`) | `j(p)=='P'` | 9 | `p>>1==4` | 7 |
| the promoted piece | `t?j(u):u` | 8 | `u*2+t` | 5 |
| the king of side `s` | `b.indexOf('kK'[+s])` | 19 | `b[Q](10+s)` | 10 |
| the enemy pawn for en passant | `b[x]=='Pp'[t]` | 13 | `b[x]==17-p` | 10 |
| White's text colour | `b[u]<U&b[u]>'-'` | 15 | `b[u]&1` | 6 |
| bishop? heavy piece? (material) | `j(p)<'C'` … `j(p)>'N'?9:j(p)=='N'` | 28 | `p<4` … `p<10?9:p>11` | 14 |
| the piece glyph | `'\xA0♝♚♞♟♛♜'['-BKNPQR'.search(j(b[u]))]` | 51 | `'\xA0♝♜♛♟♚♞'[b[u]>>1]` | 33 |

Choosing an encoding that packs colour, capability and order into one small integer is shorter than writing extra conditions around a string. It is not free everywhere, though, and the part-by-part count shows where numbers lose:

| part | letters | numbers | letters − numbers |
| --- | --- | --- | --- |
| aliases | 48 | 34 | +14 |
| state | 75 | 90 | −15 |
| `G` geometry | 297 | 273 | +24 |
| `V` attack | 64 | 50 | +14 |
| `L` legality | 95 | 110 | −15 |
| `M` make move | 230 | 218 | +12 |
| `d` draw | 287 | 269 | +18 |
| `A` apply and judge | 268 | 234 | +34 |
| everything else | 635 | 631 | +4 |
| **file** | **1,999** | **1,909** | **+90** |

- **The starting position is longer as numbers:** both boards spell themselves, but the numeric one pays for a `.map()` afterwards to turn its digits into numbers.
- **`L` differs by design, not by encoding.** On the letter board `L(i,u)` answers whether one move is legal. On the numeric board `L(i)` returns the list of legal targets, which lets `d` build the highlight once per frame (`s=L(i)`) and `M` ask `~L(x)[Q](e)`.
- **Everywhere else the letters pay:** +116 bytes across the aliases, `G`, `V`, `M`, `d` and `A` — every colour test, emptiness test and case conversion in the table above. Against the 30 bytes the numbers lose, that nets +86, and four more come from the markup: the letter file never wrote a paragraph end tag the parser does not need.

What 90 bytes buy on the letter side: a board you can read straight out of a debugger, and a repetition key that looks like the position it stands for.

## Lesson 6 — Standards, not tricks

Byte-golfed HTML usually runs on obsolete markup, because obsolete markup is shorter: `<center>`, `bgcolor`, `align`, `cellspacing`, quirks mode. Neither file does. Layout and colour are CSS; the script writes `style.background`, not `bgColor`.

**Both files validate with zero errors and zero warnings** in the W3C Nu Html Checker (version 26.9.16), CSS included. The board, the promotion picker and all 64 cells are generated at runtime, so three more documents were checked for each file: the DOM after loading, the DOM after `1.e4 e5` and a selection, and the markup strings exactly as the script writes them, before any parser repairs them. All clean.

What validity costs, per file:

| part | bytes | why it is there |
| --- | --- | --- |
| `<!DOCTYPE html>` | 15 | standards mode, where CSS box sizes mean what they say |
| `<html lang=en>` | 14 | the document language; without it the checker warns |
| `<meta charset=utf-8>` | 20 | the encoding, visible in the source and immune to editors that strip a BOM |
| `<title>C</title>` | 16 | a title is required |
| `</table>` | 8 | the table's end tag is not optional |
| `</button>` | 9 | the one end tag in the generated markup that is not optional; without it the checker reports a button opened inside a button |

Some things that look like shortcuts are not. `</td>` and `</tr>` are left out, and `<head>` and `<body>` are never written: those end tags and elements are optional by specification. Unquoted attribute values such as `onclick=S(id)` are valid wherever the value holds no space, quote, `=`, `<`, `>` or backtick; `onclick="A('q')"` in `index.html` is quoted because its value contains quotes. Numeric ids (`id=0` … `id=63`) are valid in HTML5, and the script reaches them by named access: `this[w]`.

Piece glyphs are Unicode `U+265A`–`U+265F`, recoloured with CSS for White; the picker uses `U+2655`–`U+2658`. An empty square holds a no-break space, written as the escape `\xA0` so that it stays visible in the source. No image, no font download, no CDN request.

---

## Anatomy

Every byte of both files, by part.

| part | `index.html` | `hexadecimal.html` | |
| --- | --- | --- | --- |
| markup and CSS | 277 | 277 | board, status line, picker, colours, layout |
| `<script>` tags | 17 | 17 | |
| aliases | 48 | 34 | `N` `a` `U` `j` · `N` `a` `Q` |
| state | 75 | 90 | the FEN fields |
| `z`, `R` | 20 | 20 | result code, repetition table |
| `G` | 297 | 273 | can this piece reach that square |
| `V` | 64 | 50 | is this square attacked |
| `L` | 95 | 110 | is this move legal — play it, ask, take it back |
| `C` | 27 | 27 | which castling right a square forfeits |
| `M` | 230 | 218 | clock, promotion, en passant victim, rook hop, en passant square |
| setup | 158 | 154 | promotion buttons and the 64 cells, generated |
| `d` | 287 | 269 | draw the board and the status line |
| `A` | 268 | 234 | apply the move, then the verdict |
| `S` | 91 | 93 | the click handler |
| `d()` | 3 | 3 | first draw |
| commas, semicolons, line breaks | 42 | 40 | the layout is worth its weight |
| **total** | **1,999** | **1,909** | |

Split another way: the rules — state, `z`/`R`, `G`, `V`, `L`, `C`, `M`, `A` — take **1,076** and **1,022** bytes; the page that shows them — markup, CSS, script tags, setup, `d`, `S` and the first draw — takes **833** and **813**. The rest is aliases and separators. The rulebook and the board that displays it cost about the same.

---

## Verification

The engines were checked by running them, not by reading them. Both files were driven through a DOM implementation (jsdom) and Node; the markup was checked with a local copy of the Nu Html Checker.

- **perft**, both files: starting position 20 / 400 / 8,902 at depths 1–3; Kiwipete (CPW position 2) 48 / 2,039; CPW position 3 14 / 191 / 2,812.
- **En passant**, four cases, both files: pinned capturer (square not set), legal capturer (set), no capturer (not set), a neighbour that wraps across the board edge (not set).
- **Every result code**, reached by clicking real moves: `W#` after 1.e4 e5 2.Bc4 Nc6 3.Qh5 Nf6 4.Qxf7#, `B#` after 1.f3 e5 2.g4 Qh4#, `SM` by Qg5–g6 with Kf7 against Kh8, `IM` by capturing the last pawn, `5R` on the 16th ply of knights going out and back, `75` at `M=150`.
- **Insufficient material**, ten endings after a capture: K–K, K+N–K, K+B–K, K+B–K+B on one colour and K+B+B on one colour end as `IM`; K+B–K+B on opposite colours, K+N+N–K, K+B+N–K, K+N–K+N and K+P–K play on.
- **Random self-play**, 30 games per file (10,060 and 10,472 plies), with the board array checked for corruption after every ply.
- **Rendering** compared cell by cell — glyph, colour, background, outline, status line, picker — against the previous build of each file across random games, promotions included.
- **Markup**, as described in Lesson 6.

---

## Related

- [chess1023byte](https://github.com/cuneytinann/chess1023byte) — the other end of the scale: the core rules alone, packed, in 1,023 bytes
- [fidelite.art](https://www.fidelite.art/) — the full arbiter: clock, draw offers and claims, resignation, flag fall, dead positions, fifteen result codes, eight front ends

## License

MIT

---
---

<a id="turkce"></a>

# chessarbiter2kb (Türkçe)

Tek bir HTML dosyasında **1.999 bayt** içinde yazılmış iki kişilik bir satranç hakemi ve onun **1.909 baytlık** ikizi. Kütüphane yok, derleme adımı yok, sunucu yok, paketleyici yok. Bir dosyayı indirin, çift tıklayın, oynayın.

İki dosya da aynı kuralları uygular. `index.html` tahtayı bir FEN gibi harflerle tutar; `hexadecimal.html` sayılarla. Yan yana okunduklarında, satranç kurallarının ne kadar az karakterle yazılabileceğine ve bir veri gösterimi seçiminin neye mal olduğuna dair küçük bir derstirler.

[Golfstack](https://www.fidelite.art/) projesinin bir parçasıdır.

## Oyna

| dosya | tahta | GitHub Pages | proje sitesi |
| --- | --- | --- | --- |
| `index.html` | harfler | [chessarbiter2kb](https://cuneytinann.github.io/chessarbiter2kb/) | [L2_string_flip_noBlockedDetector.html](https://www.fidelite.art/special/outofLevels/L2_string_flip_noBlockedDetector.html) |
| `hexadecimal.html` | sayılar | [hexadecimal.html](https://cuneytinann.github.io/chessarbiter2kb/hexadecimal.html) | [L2_noBlockedDetector.html](https://www.fidelite.art/special/outofLevels/L2_noBlockedDetector.html) |

Proje sitesinde iki sürüm de `special/outofLevels` altında, `L2` kural seviyesinin varyantları olarak durur.

Bir taşa, sonra hedef kareye tıklayın. Yasal hedefler kehribar rengine döner; seçili kare, ardından da son hamlenin hedef karesi yeşil çerçeveyle işaretlenir; tahta her yarım hamleden sonra sırası gelen tarafa döner.

## Tarayıcıda okuyun

Buradaki hiçbir şey bir araçla paketlenmemiş ya da küçültülmemiştir. **Sağ tık → Sayfa Kaynağını Görüntüle** (`Ctrl` `U`, macOS'ta `⌥` `⌘` `U`) programın tamamını gösterir ve bu deponun amaçlanan kullanımı da budur: sayfayı açın, birkaç hamle oynayın, sonra o hamleleri az önce yöneten kaynağı okuyun.

Betik, bağımlılık sırasına dizilmiş, virgülle ayrılmış tek bir bildirim zinciridir; her satırda bir fikir olacak şekilde bölünmüştür. İki dosya satır satır paralel dizilmiştir (56 ve 54 satır), böylece aynı kural ikisinde de aynı yüksekliktedir. Satır sonlarının bir anlamı yoktur: iki betikten de silindiğinde sözdizimi ağacı birebir aynı kalır.

```
takma adlar → durum → G → V → L → C → M → kurulum → d → A → S
              geometri · saldırı · yasallık · rok hakları · hamle yazma · çizim · uygula ve hükmet · tıklama
```

---

## İçinde neler var

- **Tüm taş hareketleri**, aritmetikten türetilmiş. Yön tablosu yok, ofset dizisi yok.
- **Tam yasallık kontrolü.** Kendi şahınızı şah altında bırakan bir hamle asla kabul edilmez.
- **Rok**, iki yöne de, tüm koşullarıyla: rok hakkı duruyor, kale yolu açık, şah şah altında değil, şah saldırı altındaki bir kareden geçmiyor, şah şah altına girmiyor.
- **Geçerken alma**, neredeyse herkesin yanlış yaptığı durum dahil (Ders 3).
- **Seçicili terfi.** Vezir, kale, fil, at. Seçim yapılana kadar tahta kilitlenir.
- **Şah, şah mat, pat** birbirinden ayırt edilir.
- **Yetersiz materyal**, FIDE'nin okumasıyla (Ders 4).
- **5 kez tekrar** ve **75 hamle kuralı**, talep gerekmeden ilan edilir.
- **Durum satırı**: köşe koordinatları, sırası gelen taraf şah altındaysa `C!`, yarım hamle sayacı `M=`, geçerli konumun kaç kez oluştuğu `R=`. Oyun bittiğinde satır sonuç koduna dönüşür.

## İçinde neler yok

- Saat yok.
- Beraberlik teklifi ve beraberlik talebi yok; dolayısıyla 3 kez tekrar talebi ve 50 hamle talebi de yok. Sayaçlar çalışır ve gösterilir; oyunu yalnızca otomatik eşikler bitirir.
- **Kilitli pozisyon** algılaması yok. Bu, FIDE 5.2.2 maddesinin diğer yarısıdır: materyal yeterlidir ama piyonlar kilitlenmiştir ve mat zaten imkânsızdır. FideLite'ın `L2` sürümlerinde bu dedektör (`J`) sayısal tahtada 457, harf tahtasında 702 bayttır ve bilerek çıkarılmıştır. Bu yüzden sonuç kodu `DP` (ölü pozisyon) değil `IM`'dir (yetersiz materyal): kod, gerçekte neyin test edildiğini söyler. Kilitli bir pozisyon yine beraberlikle biter, yalnızca daha geç: tekrarla ya da 75 hamle kuralıyla.
- Bot yok, geri alma yok, FEN içe ya da dışa aktarma yok, PGN yok.

Eksiksiz hakem için — saat, beraberlik teklifleri ve talepleri, terk, süre bitimi, ölü pozisyonlar, on beş sonuç kodu — [fidelite.art](https://www.fidelite.art/) adresine bakın.

## Sonuç kodları

| kod | anlamı |
| --- | --- |
| `W#` | Beyaz mat eder |
| `B#` | Siyah mat eder |
| `SM` | Pat |
| `IM` | Yetersiz materyal |
| `5R` | 5 kez tekrar |
| `75` | 75 hamle kuralı |

---

## Ders 1 — Durum bir FEN kaydıdır

```js
b=[...`rnbqkbnrpppppppp${(_='-').repeat(32)}PPPPPPPPRNBQKBNR`],t=1,c=15,e=_,n=0,   // index.html
b=[...'5d37b3d599999999'+'0'.repeat(32)+'888888884c26a2c4'].map(u=>'0x'+u-0),t=1,c=15,e=-1,n=0,  // hexadecimal.html
```

İki satır da alan alan ve FEN'in kendi sırasıyla bir FEN'dir: **taş dizilimi, sırası gelen taraf, rok hakları, geçerken alma karesi, yarım hamle sayacı.** Dışarıda kalan tek alan tam hamle numarasıdır, çünkü onu hiçbir kural okumaz: 50 ve 75 hamle kuralları yarım hamle sayar, tekrar ise hamle numarasına göre değil konuma göre belirlenir.

- **Harfler.** 0 numaralı eleman a8'dir; FEN de bu sırayla yazılır. Büyük harf Beyaz, küçük harf Siyah, `-` boş karedir.
- **Sayılar.** 0 numaralı eleman a1'dir. Her kare tek bir onaltılık rakam tutar: `0` boş, gerisi `tür×2 + renk`. İki dosya da altmış dört karakterini aynı biçimde ve aynı üç parçada kurar: bir taraf için on altı, boş orta için `repeat(32)`, öteki taraf için on altı. İki satırı yan yana okuyun, aynı kare ikisinde de aynı konumda durur.

Sayısal tahta ortasını eskiden `10n**40n-10n**32n` diye yazıyordu; `99999999` ve ardından otuz iki sıfır veren bir BigInt — altı bayt daha kısa, ve okuyan için tam bir duraklama. Bilerek açıldı. Bu dosyalar okunmak için yazıldı, ve görülebilmesi için önce hesaplanması gereken bir tahta, onu göstermenin amacını ortadan kaldırır.

Tekrar anahtarı iki dosyada da aynı dört değişkenin birleştirilmesidir:

```js
$=R[s=b+t+e+c]=-~R[s]     // tahta, sırası gelen taraf, geçerken alma karesi, rok hakları
```

`-~undefined` değeri `1`'dir; bu yüzden ilk kez görülen bir konum için varsayılan değer yazmak gerekmez. Harf tahtasında anahtar yazdırıldığında neredeyse bir FEN gibi okunur.

## Ders 2 — Geometri aritmetiktir

`G(i,f)` tek bir soruyu yanıtlar: `i` karesindeki taş, tahtadan başka hiçbir şeye bakmadan `f` karesine ulaşabilir mi? Her şeyi iki sayıdan türetir: dikey sıra farkı `h` ve yatay sıra farkı `v`.

```js
h=a(i%8-f%8),v=a(y-(f>>3))     dikey ve yatay sıra farkı
d=h|v                          doğrusal bir hamlenin kat ettiği kare sayısı: h==0, v==0 ya da h==v iken h|v, max(h,v)'dir
k=(f-i)/d                      adım vektörü — ±1, ±8, ±7, ±9 — tek bir bölmeyle
h*v==2                         at: 2 sonucunu yalnızca 1×2 ve 2×1 verir
T|v|h^2?d<2:…                  şah: bir kare ya da rok olarak yatayda iki kare
i+3.5*k-.5                     rok kalesi: şah kanadında i+3, vezir kanadında i-4, tek ifade
h*v==1                         piyon alışı: bir dikey, bir yatay
66>>y                          piyonların başlangıç yatayları: 66 = 0b1000010, yalnız 1. ve 6. bitler açık
f%56<8                         son yatay, iki renk için de
f^8                            geçerken alınan piyon: 8 ile XOR, doğru yönde bir yatay geri gider
(i/8^i)%2                      bir karenin rengi: yatay paritesi XOR dikey paritesi
```

Uzun menzilli taşlar hattı `S` ile adım adım yürür; hedefe ya da ilk dolu kareye gelince durur. Hangi taşın hangi hatta yürüyebileceği, iki dosyanın ilk ayrıştığı yerdir ve ikisini de okumaya değer:

```js
(h*v?h==v&P<'R':P>'B')&S()     harfler: alfabede B < Q < R; P<'R' fil ve veziri çaprazda,
                               P>'B' vezir ve kaleyi düz hatta geçirir
(h*v?h==v:2)&P&&S()            sayılar: fil=1, kale=2, vezir=3; tür kodu kendisi bir bit maskesidir;
                               çapraz 1, düz hat 2 verir, gerisini & yapar
```

İşlemcide ikinci bir ders saklıdır. Harf tahtasında kenarın ötesinden okunan değer `undefined`'dır ve `'-'` değildir; bu yüzden hat olmayan bir yürüyüş kendiliğinden durur ve `&` güvenlidir. Sayısal tahtada `!undefined` değeri `true`'dur; yürüyüş `&&` ile korunmalıdır: `S` ancak yetenek testi gerçek bir hattı doğruladıktan sonra çalışır. O durumda adım tam sayıdır ve ziyaret edilen her kare tahtanın içindedir.

## Ders 3 — Yasallık: oyna, sor, geri al

Yasallık ikinci bir hamle üreticisi değildir, tek bir numaradır:

- `V(s)`, bir karenin saldırı altında olup olmadığını bütün tahtaya sorar; varsayılan olarak `s` tarafının şahını.
- `L`, aday hamleyi kopyalanmış bir tahtada `M` ile oynar, `V`'ye sorar ve tahtayı geri yükler.
- `M` bir hamleyi yazar ve başka hiçbir şey yapmaz, çünkü `L` onu deneme olarak çağırır. Kalıcı bir şey yapan tek fonksiyon `A`'dır.

Bağımlılık tam iki yerde geriye doğru akar ve ikisi de koddan değil kurallardan gelir. `G`, `V`'yi çağırır; çünkü rok, şahın geçtiği karenin saldırı altında olup olmadığı sorulmadan değerlendirilemez. `M`, `L`'yi çağırır; çünkü geçerken alma karesi ancak alış gerçekten yasalsa yazılabilir:

```js
e=P&d>9&&(e=q,[f-1,f+1].some(x=>b[x]=='Pp'[t]&&L(x,e)))?e:_      // harfler
e=P&d>9&&(e=q,[f-1,f+1].some(x=>b[x]==17-p&&~L(x)[Q](e)))?e:-1   // sayılar: 17-p karşı tarafın piyonu
```

Bir piyon az önce iki kare ilerledi. Saf uygulama geçerken alma karesini hemen yazar. Bu uygulama kareyi geçici olarak yazar, komşu bir rakip piyonun oraya *yasal olarak* alış yapıp yapamayacağını sorar, yapamıyorsa geri alır.

Fark alışın kendisinde hiç görünmez: yasadışı bir alışı hamle üreticisi zaten reddeder. Fark **tekrar sayacında** görünür, çünkü `e` konum anahtarının parçasıdır. FIDE iki konumu ancak ikisinde de aynı hamleler — geçerken alma dahil — mümkünse aynı sayar. Kimsenin kullanamayacağı bir kare yazarsanız aynı konum iki farklı anahtar altına düşer: tekrar beraberliği geç gelir ya da hiç gelmez.

Klasik örnek: Siyah g7–g5 oynar; f5'teki beyaz piyon, f8'deki kale tarafından f1'deki şahına açmazlanmıştır. `fxg6` şahı açığa çıkarır, alış yasadışıdır, kare hiç yazılmaz. Bu kontrol, saf `e=P&d>9?q:_` biçimine göre harf tahtasında **47**, sayısal tahtada **49 bayt** tutar. `[f-1,f+1]`'in kenar durumunu da kendiliğinden çözer: tahtanın öbür kenarına sarılan bir komşu dizindeki piyonu `G` reddeder ve kare yazılmadan kalır.

## Ders 4 — Herkesin atladığı kurallar ucuzdur

Kod golfü koleksiyonlarındaki satranç programları yarım hamle sayacını, tekrar sayacını ve yetersiz materyal testini pahalı defter tutma diye sıklıkla çıkarır. Bu dosyalarda ölçülen değerler:

| kural | harfler | bayt | sayılar | bayt |
| --- | --- | --- | --- | --- |
| Yarım hamle sayacı | `n=P\|b[f]>_?0:n+1` | 16 | `n=P\|b[f]?0:n+1` | 14 |
| Tekrar sayacı | `$=R[s=b+t+e+c]=-~R[s]` | 21 | aynı | 21 |
| Dört rok hakkının hepsi | `C=i=>'20003001'[i%56]<<i/28` | 27 | aynı | 27 |
| Yetersiz materyal, iki taraf birden | `(m=W=0,b.map((p,i)=>p>_&&(j(p)<'C'?m\|=(i/8^i)%2+1:W+=j(p)>'N'?9:j(p)=='N')),W*2+m<3)` | 84 | `(m=W=0,b.map((p,i)=>p&&(p<4?m\|=(i/8^i)%2+1:W+=p<10?9:p>11)),W*2+m<3)` | 68 |
| Oyunun bütün bitişleri | `?'IM':$>4?'5R':n>149&&'75':V(t)?t?'B#':'W#':'SM'` | 48 | `?'IM':$>4?'5R':n>149?'75':0:V(t)?'WB'[t]+'#':'SM'` | 49 |
| **toplam** | | **196** | | **179** |

İki dosyanın da onda birinden az. Bir satranç programının pahalı kısmı hiçbir zaman kural kitabı olmadı.

**Rok hakları** tek bir dizgedir: dizinin ilk ya da son yatayındaki bir karenin altındaki rakam, o kareden bir taş ayrıldığında ya da oraya bir taş geldiğinde kaybedilen haktır; `i/28` kadar kaydırılır — dizinin bir ucunda 0, öbür ucunda 2. Şah ve kale kareleri kendi bitlerini taşır; diğer bütün kareler `0` verir ve `A` her hamlenin iki karesini `c&=~C(i)&~C(f)` ile temizler.

**Yetersiz materyal** tek geçiş ve tek karşılaştırmadır. `m`, fillerin durduğu kare renklerini bit olarak toplar (1 ya da 2; iki renk birden 3 eder). `W` bir atı 1, herhangi bir piyon, kale ya da veziri 9 sayar; şahlar bir şey eklemez. Böylece `W*2+m<3` tam olarak şu durumlarda doğrudur: yalnız şahlar, tek bir at ya da hepsi aynı renk karede duran filler — hangi tarafta olursa olsun. İki at, fil ve at, zıt renkli filler ya da her iki tarafta birer at 3 ya da daha fazla verir ve oyun sürer.

**Talepler ve ilanlar.** *Talep* edilmesi gereken bir beraberlik, onu talep edecek bir oyuncuya, o da bir düğmeye ihtiyaç duyar. Bu sürümde böyle bir katman yok; bu yüzden yalnızca bir hakemin kendiliğinden ilan ettiği bitişler tutulur: 5 kez tekrar, 75 hamle kuralı, yetersiz materyal, mat ve pat. `R=`'nin 3'e çıktığını, `M=`'nin 99'u geçtiğini yine de izleyebilirsiniz — FIDE'ye göre bu iki eşik ilan değil taleptir, bu yüzden burada oyunu bitirmezler.

## Ders 5 — Harfler mi, sayılar mı

Bir harf tek bir şey söyler: hangi taş. Renk harfin büyüklüğünden, boşluk ayrı bir karakterden çıkarılmak zorundadır; bu yüzden neredeyse her soru fazladan bir karşılaştırma ya da büyük harfe çevirme ister (`j`, `toUpperCase`'tir). Bir sayı ise soruları bitleri zaten yanıtlayacak şekilde seçilebilir:

| kod | 0 | 2 · 3 | 4 · 5 | 6 · 7 | 8 · 9 | a · b | c · d |
| --- | --- | --- | --- | --- | --- | --- | --- |
| taş | boş | fil | kale | vezir | piyon | şah | at |

0. bit renktir (1 Beyaz). Tür, yani `p>>1`, şöyle sıralanır: fil 1, kale 2, vezir 3, piyon 4, şah 5, at 6. Bu sıra alfabetik değil, işlevseldir: uzun menzilli taşların türleri aynı zamanda yön maskesidir (Ders 2) ve `G` düz karşılaştırmalarla dallanabilir — `P>5` at, `P>4` şah, `P>3` piyon, geri kalanlar kayar.

Aynı sorular, iki tahtaya ayrı ayrı:

| soru | harfler | bayt | sayılar | bayt |
| --- | --- | --- | --- | --- |
| boş kare | `b[i]==_` | 7 | `!b[i]` | 5 |
| dolu kare | `b[f]>_` | 6 | `b[f]` | 4 |
| `s` tarafına saldırabilecek aday (`V` içinde) | `p>_&p<U!=s` | 10 | `p&1^s` | 5 |
| at mı | `P=='N'` | 6 | `P>5` | 3 |
| piyon mu (`M` içinde) | `j(p)=='P'` | 9 | `p>>1==4` | 7 |
| terfi eden taş | `t?j(u):u` | 8 | `u*2+t` | 5 |
| `s` tarafının şahı | `b.indexOf('kK'[+s])` | 19 | `b[Q](10+s)` | 10 |
| geçerken alabilecek rakip piyon | `b[x]=='Pp'[t]` | 13 | `b[x]==17-p` | 10 |
| Beyaz'ın yazı rengi | `b[u]<U&b[u]>'-'` | 15 | `b[u]&1` | 6 |
| fil mi? ağır taş mı? (materyal) | `j(p)<'C'` … `j(p)>'N'?9:j(p)=='N'` | 28 | `p<4` … `p<10?9:p>11` | 14 |
| taş karakteri | `'\xA0♝♚♞♟♛♜'['-BKNPQR'.search(j(b[u]))]` | 51 | `'\xA0♝♜♛♟♚♞'[b[u]>>1]` | 33 |

Rengi, yeteneği ve sırayı tek bir küçük tam sayıya yükleyen bir kodlama seçmek, bir dizgenin etrafına fazladan koşullar yazmaktan daha kısadır. Ama her yerde bedava değildir; parça parça sayım sayıların nerede kaybettiğini gösterir:

| parça | harfler | sayılar | harfler − sayılar |
| --- | --- | --- | --- |
| takma adlar | 48 | 34 | +14 |
| durum | 75 | 90 | −15 |
| `G` geometri | 297 | 273 | +24 |
| `V` saldırı | 64 | 50 | +14 |
| `L` yasallık | 95 | 110 | −15 |
| `M` hamle yazma | 230 | 218 | +12 |
| `d` çizim | 287 | 269 | +18 |
| `A` uygula ve hükmet | 268 | 234 | +34 |
| geri kalan her şey | 635 | 631 | +4 |
| **dosya** | **1.999** | **1.909** | **+90** |

- **Başlangıç konumu sayılarla daha uzundur:** iki tahta da kendini heceler, ama sayısal olan rakamlarını sayıya çevirmek için arkasından bir `.map()` öder.
- **`L` kodlama yüzünden değil, tasarım yüzünden farklıdır.** Harf tahtasında `L(i,u)` tek bir hamlenin yasal olup olmadığını yanıtlar. Sayısal tahtada `L(i)` yasal hedeflerin listesini döndürür; bu da `d`'nin vurgulamayı her çizimde bir kez kurmasını (`s=L(i)`) ve `M`'nin `~L(x)[Q](e)` diye sormasını sağlar.
- **Diğer her yerde harfler öder:** takma adlar, `G`, `V`, `M`, `d` ve `A` boyunca +116 bayt — yukarıdaki tablodaki her renk testi, boşluk testi ve büyük harfe çevirme. Sayıların kaybettiği 35 bayt düşülünce net fark +87 olur; dört bayt daha işaretlemeden gelir: harf dosyası, ayrıştırıcının zaten gerek duymadığı bir paragraf kapanış etiketini hiç yazmamıştı.

Harf tarafında 90 baytın karşılığı: bir hata ayıklayıcıdan doğrudan okunabilen bir tahta ve temsil ettiği konuma benzeyen bir tekrar anahtarı.

## Ders 6 — Hile değil, standart

Bayt golfü yapılmış HTML genellikle eskimiş işaretlemeyle çalışır, çünkü eskimiş işaretleme daha kısadır: `<center>`, `bgcolor`, `align`, `cellspacing`, quirks kipi. İki dosya da bunların hiçbirini kullanmaz. Yerleşim ve renk CSS'tedir; betik `bgColor` değil `style.background` yazar.

**İki dosya da W3C Nu Html Checker'da (sürüm 26.9.16), CSS dahil, sıfır hata ve sıfır uyarıyla doğrulanır.** Tahta, terfi seçicisi ve 64 hücrenin hepsi çalışma anında üretildiği için her dosya için üç belge daha denetlendi: yüklemeden sonraki DOM, `1.e4 e5` ve bir seçimden sonraki DOM, ve betiğin yazdığı işaretleme dizgelerinin, ayrıştırıcı onları onarmadan önceki birebir hâli. Hepsi temiz.

Geçerliliğin dosya başına maliyeti:

| parça | bayt | neden orada |
| --- | --- | --- |
| `<!DOCTYPE html>` | 15 | standart kipi; CSS kutu boyutları söylediği anlama gelir |
| `<html lang=en>` | 14 | belgenin dili; yazılmazsa denetleyici uyarır |
| `<meta charset=utf-8>` | 20 | kodlama; kaynakta görünür ve BOM'u silen editörlerden etkilenmez |
| `<title>C</title>` | 16 | başlık zorunludur |
| `</table>` | 8 | tablonun bitiş etiketi isteğe bağlı değildir |
| `</button>` | 9 | üretilen işaretlemedeki isteğe bağlı olmayan tek bitiş etiketi; yazılmazsa denetleyici düğme içinde açılmış bir düğme bildirir |

Kısayol gibi görünen bazı şeyler kısayol değildir. `</td>` ve `</tr>` yazılmaz, `<head>` ve `<body>` hiç yazılmaz: bu bitiş etiketleri ve öğeler spesifikasyona göre isteğe bağlıdır. `onclick=S(id)` gibi tırnaksız öznitelik değerleri, değer boşluk, tırnak, `=`, `<`, `>` ya da ters tırnak içermediği sürece geçerlidir; `index.html`'deki `onclick="A('q')"` ise değeri tırnak içerdiği için tırnaklıdır. Sayısal id'ler (`id=0` … `id=63`) HTML5'te geçerlidir ve betik onlara isimli erişimle ulaşır: `this[w]`.

Taş karakterleri Unicode `U+265A`–`U+265F` aralığındadır ve Beyaz için CSS ile yeniden renklendirilir; seçici `U+2655`–`U+2658` kullanır. Boş kare bölünmez bir boşluk tutar; kaynakta görünür kalsın diye `\xA0` kaçış dizisiyle yazılmıştır. Görsel yok, yazı tipi indirme yok, CDN isteği yok.

---

## Anatomi

İki dosyanın her baytı, parça parça.

| parça | `index.html` | `hexadecimal.html` | |
| --- | --- | --- | --- |
| işaretleme ve CSS | 277 | 277 | tahta, durum satırı, seçici, renkler, yerleşim |
| `<script>` etiketleri | 17 | 17 | |
| takma adlar | 48 | 34 | `N` `a` `U` `j` · `N` `a` `Q` |
| durum | 75 | 90 | FEN alanları |
| `z`, `R` | 20 | 20 | sonuç kodu, tekrar tablosu |
| `G` | 297 | 273 | bu taş o kareye ulaşabilir mi |
| `V` | 64 | 50 | bu kare saldırı altında mı |
| `L` | 95 | 110 | bu hamle yasal mı — oyna, sor, geri al |
| `C` | 27 | 27 | bir karenin hangi rok hakkını kaybettirdiği |
| `M` | 230 | 218 | sayaç, terfi, geçerken alınan piyon, kale atlaması, geçerken alma karesi |
| kurulum | 158 | 154 | terfi düğmeleri ve 64 hücre, üretilmiş |
| `d` | 287 | 269 | tahtayı ve durum satırını çiz |
| `A` | 268 | 234 | hamleyi uygula, sonra hükmü ver |
| `S` | 91 | 93 | tıklama işleyicisi |
| `d()` | 3 | 3 | ilk çizim |
| virgüller, noktalı virgüller, satır sonları | 42 | 40 | bu yerleşim maliyetine değer |
| **toplam** | **1.999** | **1.909** | |

Başka bir açıdan bölünce: kurallar — durum, `z`/`R`, `G`, `V`, `L`, `C`, `M`, `A` — **1.076** ve **1.022** bayt tutar; onları gösteren sayfa — işaretleme, CSS, betik etiketleri, kurulum, `d`, `S` ve ilk çizim — **833** ve **813**. Geri kalanı takma adlar ve ayraçlardır. Kural kitabı ile onu gösteren tahta aşağı yukarı aynı tutar.

---

## Doğrulama

Motorlar okunarak değil çalıştırılarak denetlendi. İki dosya da bir DOM uygulaması (jsdom) ve Node üzerinde sürüldü; işaretleme, Nu Html Checker'ın yerel bir kopyasıyla denetlendi.

- **perft**, iki dosyada da: başlangıç konumunda 1–3 derinlikte 20 / 400 / 8.902; Kiwipete (CPW 2. konum) 48 / 2.039; CPW 3. konum 14 / 191 / 2.812.
- **Geçerken alma**, dört durum, iki dosyada da: açmazdaki alıcı (kare yazılmaz), yasal alıcı (yazılır), alıcı yok (yazılmaz), tahtanın kenarından öbür tarafa sarılan komşu (yazılmaz).
- **Her sonuç kodu**, gerçek hamleler tıklanarak elde edildi: 1.e4 e5 2.Bc4 Nc6 3.Qh5 Nf6 4.Qxf7# ile `W#`, 1.f3 e5 2.g4 Qh4# ile `B#`, Kh8'e karşı Kf7 varken Qg5–g6 ile `SM`, son piyonu alarak `IM`, atların çıkıp geri döndüğü 16. yarım hamlede `5R`, `M=150`'de `75`.
- **Yetersiz materyal**, bir alıştan sonra kalan on oyun sonu: Ş–Ş, Ş+A–Ş, Ş+F–Ş, aynı renkte Ş+F–Ş+F ve aynı renkte Ş+F+F `IM` ile biter; zıt renkte Ş+F–Ş+F, Ş+A+A–Ş, Ş+F+A–Ş, Ş+A–Ş+A ve Ş+P–Ş sürer.
- **Rastgele kendi kendine oyun**, dosya başına 30 oyun (10.060 ve 10.472 yarım hamle); her yarım hamleden sonra tahta dizisi bozulmaya karşı denetlendi.
- **Görüntü**, her dosyanın bir önceki sürümüyle rastgele oyunlarda, terfiler dahil, hücre hücre karşılaştırıldı: taş karakteri, renk, arka plan, çerçeve, durum satırı, seçici.
- **İşaretleme**, Ders 6'da anlatıldığı gibi.

---

## İlgili

- [chess1023byte](https://github.com/cuneytinann/chess1023byte) — ölçeğin öbür ucu: yalnızca temel kurallar, paketlenmiş, 1.023 bayt
- [fidelite.art](https://www.fidelite.art/) — eksiksiz hakem: saat, beraberlik teklifleri ve talepleri, terk, süre bitimi, ölü pozisyonlar, on beş sonuç kodu, sekiz ön yüz

## Lisans

MIT
