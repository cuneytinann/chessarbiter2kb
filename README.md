**[English](#chessarbiter2kb)** · **[Türkçe](#turkce)**

# chessarbiter2kb

A two-player chess arbiter in **2,047 bytes** of one HTML file. No libraries, no build step, no server. Download `index.html`, double-click, play.

The board is a FEN string, not a bit set. The markup is valid HTML5 with no obsolete elements or attributes. Both of those are the point of this build.

Part of the [Golfstack](https://www.fidelite.art/) project.

## Play

- [cuneytinann.github.io/chessarbiter2kb](https://cuneytinann.github.io/chessarbiter2kb/)
- [fidelite.art/special/outofLevels/L2_string_flip_noBlockedDetector.html](https://www.fidelite.art/special/outofLevels/L2_string_flip_noBlockedDetector.html) — same file, mirrored on the project site among the `L2` builds

The name of the budget: 2,048 bytes. This lands 1 byte under it.

Click a piece, click a destination. Legal targets turn amber, the last move keeps a green outline, and the board flips to the side to move after every ply.

---

## What it is for

This build exists to be read on a slide, not to win a byte record. It makes three claims, and every one of them is a number you can check in the source.

**1. The rules everyone skips are the cheap ones.** Chess programs in code-golf collections routinely drop the halfmove clock, the repetition counter and the insufficient-material test, on the grounds that they are bookkeeping and bookkeeping is expensive. They are not expensive:

| rule | the whole implementation | bytes |
| --- | --- | --- |
| Halfmove clock (`M=`) | `n=P\|b[f]>_?0:n+1` | **16** |
| Repetition counter (`R=`) | `$=R[s=b+t+e+c]=-~R[s]` | **21** |
| Castling rights, all four bits | `C=i=>'20003001'[i%56]<<i/28` | **27** |
| Insufficient material, both sides at once | `(m=W=0,b.map((p,i)=>p>_&&(q=j(p),q<'C'?m\|=(i/8^i)%2+1:W+=q>'N'?9:q=='N')),W*2+m<3)` | **82** |
| Every ending the game can have | `?'IM':$>4?'5R':n>149&&'75':l(t)?t?'B#':'W#':'SM'` | **48** |

Five rules, 194 bytes, a tenth of the file. The expensive part of a chess program was never the rulebook.

**2. A readable board costs almost nothing.** The array holds the same letters a FEN holds — `rnbqkbnr`, `P`, `-` — instead of numeric piece codes. Measured against the numeric variant of the same engine, function for function and with the same rule set on both sides, the letters cost **74 bytes**: 805 against 880 across the state and the six rule functions, under 4% of the file. What they buy is a board you can read straight out of a debugger, and a repetition key that looks like the position it stands for.

**3. Only the rules that fire by themselves.** A draw that has to be *claimed* needs a player to claim it, which needs a button, which needs an interaction layer. This build has no such layer, so it keeps only the endings an arbiter declares on its own: 5-fold repetition, the 75-move rule, insufficient material, mate, stalemate. The counters still run and are still displayed — you can watch `R=` climb to 3 and `M=` pass 99 — they simply do not end the game here, because under FIDE those two thresholds are claims, not declarations.

---

## The board is a FEN record

```js
b=[...'rnbqkbnrpppppppp'+_.repeat(32)+'PPPPPPPPRNBQKBNR'],t=1,c=15,e=_,n=0,
```

That single line is a FEN, field for field, in FEN's own order: **board, side to move, castling rights, en passant square, halfmove clock.** Index 0 is a8 and index 63 is h1, which is the order a FEN is written in and the order a board is read in. Uppercase is White, lowercase is black, `-` is an empty square.

The only FEN field that is missing is the fullmove number, and it is missing because no rule reads it. The 50- and 75-move rules count plies; repetition is decided by a position key, not a move number.

What this representation buys, everywhere in the engine:

```js
b[i]              the square             one array access
j(b[i])           the piece type         'P', 'N', 'K' …
b[i]<E            the colour             uppercase is White
b[i]==_           empty
b+t+e+c           the repetition key     board, side, ep square, rights
```

The last line is worth a second look: the key FIDE uses to decide whether two positions are the same is a string concatenation of four variables, and it reads almost like a FEN when you print it.

---

## Anatomy

Every byte of the file, by part.

| part | bytes | |
| --- | --- | --- |
| markup and CSS | 252 | board, status line, colours, layout |
| aliases | 59 | `_` `N` `E` `a` `j` |
| state | 75 | the FEN line above |
| `z`, `R` | 22 | result code, repetition table |
| `G` | 293 | geometry: can this piece reach that square |
| `V` | 45 | is this square attacked |
| `l` | 30 | is this king in check |
| `L` | 95 | is this move legal — play it, ask, take it back |
| `C` | 28 | which castling right a square forfeits |
| `M` | 233 | make the move: clock, promotion, ep victim, rook hop, ep square |
| setup | 163 | promotion buttons and the 64 cells, generated |
| `d` | 274 | draw the board and the status line |
| `A` | 257 | apply the move, then the verdict |
| `S` | 95 | the click handler |
| BOM, `<script>` tags, line breaks | 126 | the layout below is worth its weight |

Split another way: **928 bytes of chess, 1,119 bytes of everything else.** The rules cost about the same as the board that displays them.

---

## What's in it

- **All piece movement**, derived from arithmetic. No direction tables, no offset arrays. The knight is `h*v==2`; the sliding pieces read their own type letter as a capability test.
- **Full legality.** A move that leaves your own king in check is never accepted. Every candidate is played on a cloned board and the king is queried.
- **Castling**, both sides, with every condition: right still held, rook path clear, king not in check, king not crossing an attacked square, king not landing in check.
- **En passant**, including the part almost everyone gets wrong — see below.
- **Promotion with a picker.** Queen, rook, bishop, knight. The board locks until you choose, and the move is not completed until then.
- **Check, checkmate, stalemate**, told apart.
- **Insufficient material** to FIDE's reading: bare kings, a lone knight, a lone bishop, and bishops that all stand on one square colour are dead; opposite-coloured bishops, two knights and bishop-plus-knight are not.
- **5-fold repetition** and the **75-move rule**, declared without a claim.
- **Board flip** to the side to move, after every ply.
- **Status line**: corner coordinates, `C!` when the side to move is in check, `M=` the halfmove clock, `R=` how many times the current position has been seen. When the game ends, the line becomes the result code.

## What's not in it

- No clock.
- No draw offer and no draw claim, so no 3-fold claim and no 50-move claim. The counters run and are shown; only the automatic thresholds end the game.
- No detection of **blocked positions** — the other half of FIDE 5.2.2, where the material is sufficient but the pawns have locked and mate is impossible anyway. That detector is 497 bytes, a quarter of a file this size, and it was removed deliberately; the result code here is therefore `IM` (insufficient material) rather than `DP` (dead position), because the code names what is actually tested. A blocked position still ends as a draw, just later, by repetition or by the 75-move rule.
- No bot, no undo, no FEN import or export, no PGN.

For the full FIDE arbiter — clock, draw offers and claims, resignation, flag fall, dead positions, fifteen result codes — see [fidelite.art](https://www.fidelite.art/).

## Result codes

Six endings, two characters each.

| code | meaning |
| --- | --- |
| `W#` | White delivers mate |
| `B#` | Black delivers mate |
| `SM` | Stalemate |
| `IM` | Insufficient material |
| `5R` | 5-fold repetition |
| `75` | 75-move rule |

---

## Reading the source

The file is one comma-separated declaration chain, in dependency order. Each function leans only on the ones before it — and it is broken across 56 lines so that one line holds one rule. The four piece families sit on four consecutive lines. A move is written in five lines, in the order those five steps have to happen. Every way the game can end is a four-line ladder. None of that costs anything but whitespace: strip every line break and the file is byte for byte what it was.

```
aliases  →  state  →  G  →  V  →  l  →  L  →  C  →  M  →  driver
            geometry ── attack ── legality ── application
```

`G` asks whether a piece can reach a square, ignoring everything else. `V` asks the whole board whether a square is attacked. `l` asks it about a king. `L` settles legality the only way it can be settled — it plays the move on a cloned board, asks `l`, and puts the board back. `M` writes a move and nothing else, because `L` calls it as a trial. `A` is the only function that makes anything permanent.

There are two places where the dependency runs backwards, and both come from the chess rules rather than from the code. `G` calls `V` because castling cannot be judged without asking whether the square the king crosses is attacked. `M` calls `L` because the en passant square may only be written if the capture is genuinely legal.

### Selected lines

```js
d=h|v                      how many squares the move spans, without a branch
k=(f-i)/d                  the step vector: ±1, ±8, ±7, ±9, from one division
h*v==2                     the knight: the only integer pairs giving 2 are 1×2 and 2×1
y%5==1                     both pawn home ranks — the only ranks with remainder 1 mod 5
f%56<8                     the last rank, either colour
f^8                        the en passant victim, direction found by the XOR itself
i+3.5*k-.5                 the castling rook's square, kingside and queenside in one expression
h==v&P<'R'                 diagonal movers: bishop and queen pass, rook fails, by alphabet
P>'B'                      straight movers: rook and queen pass, bishop fails
W*2+m<3                    the entire insufficient-material table, one comparison
-~R[s]                     a position seen for the first time lands on 1, no default write
'-BKNPQR'.search(j(b[u]))  letter to glyph, no map, no switch
```

The sliding pieces are the neatest of these. `B < Q < R` in the alphabet, so `P<'R'` is exactly "may move diagonally" and `P>'B'` is exactly "may move in a straight line". The queen needs no branch of its own; it is simply the letter that passes both tests.

### The en passant subtlety

This is the most instructive bug in the whole rulebook, and it is the reason `M` reaches back into the legality layer:

```js
e=P&d>9&&(e=q,[f-1,f+1].some(x=>b[x]=='Pp'[+g]&&L(x,e)))?e:_
```

A pawn just moved two squares. The naive implementation writes the en passant square immediately. This one writes it provisionally, then checks whether a neighbouring enemy pawn can *actually and legally* make that capture — and reverts to `-` if it cannot.

The difference never shows up in the capture itself: if the capture is illegal, the move generator rejects it either way. It shows up in the **repetition counter**, because `e` is part of the position key. FIDE counts two positions as identical only if the same moves are available in both, en passant included. Write a square nobody can use and the same position lands under two different keys, and a repetition draw fires late or never fires at all. A game that should have been drawn can be lost that way.

The classic case: Black plays g7–g5 and the white pawn on f5 is pinned against its king by a rook on f8. `fxg6` would open the file and expose the king, so the capture is not legal, so the square is never set. 60 bytes, and they are the difference between a counter that is correct and one that is merely plausible.

---

## Standards, not tricks

Byte-golfed HTML usually runs on obsolete markup, because obsolete markup is shorter: `<center>`, `bgcolor`, `align` and `width` on cells, `cellspacing`, quirks mode. This build does none of that. Every one of those was measured, and then moved into CSS.

The file **validates with zero errors and zero warnings** — checked with the W3C Nu validator, not asserted. And not only the file: since the board, the promotion picker and all 64 cells are generated at runtime, a snapshot of the DOM the page actually builds was validated as a separate document. Both come back clean.

| | |
| --- | --- |
| obsolete elements | none |
| obsolete attributes | none |
| quirks mode | no — standards mode is kept deliberately |
| cost of all of the above | **45 bytes**, against the same build written in legacy markup |

Some things that look like shortcuts are not. `</p>`, `</td>`, `</tr>` and `</html>` are omitted, and `<html>`, `<head>` and `<body>` are never written as elements — all of that is valid HTML, the end tags are optional by specification. `</button>` **is** written, because that one is not optional. Unquoted attribute values are valid too, wherever the value carries no space or quote.

The encoding is declared by a **UTF-8 BOM** rather than a `<meta charset>` element: three bytes against twenty, and the BOM is step one of the HTML encoding sniffing algorithm, ahead of both the meta element and the HTTP header. The validator accepts it as the declaration. If you edit this file, make sure your editor preserves it, or the piece glyphs will break.

Standards mode is not sentiment either. In quirks mode `width` would include the cell's padding, the cells would come out 66 wide and 68 tall, and the board would stop being square. The doctype costs 15 bytes and keeps it square.

Piece glyphs are Unicode `U+265A`–`U+265F`, recoloured with CSS for White. No image, no font download, no CDN request.

---

## Verification

The engine was checked by running it, not by reading it.

- **perft** from the starting position: 20 / 400 / 8,902 at depths 1–3. Kiwipete (CPW position 2): 48 / 2,039. CPW position 3: 14 / 191 / 2,812.
- **En passant**, all four cases: pinned capturer (square not set), legal capturer (set, and present in the repetition key), no capturer, capturer on the wrong file.
- **Every result code** reproduced from a real position — mate, stalemate, insufficient material with each material combination, 5-fold repetition at 16 plies, the 75-move rule at `M=150`.
- **Rendering** compared square by square — glyph, colour, background, outline, status line — against the previous build across a full game, at every selection and every ply.
- **Random self-play** to termination, repeatedly, with the board array checked for corruption after every ply.
- **Markup** validated with the W3C Nu validator, both the source file and a snapshot of the runtime DOM.

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

Tek bir HTML dosyasında, **2.047 bayt** içinde yazılmış iki kişilik bir satranç hakemi. Kütüphane yok, derleme adımı yok, sunucu yok. `index.html` dosyasını indirin, çift tıklayın, oynayın.

Tahta bir bit kümesi değil, bir FEN dizesidir. İşaretleme, eskimiş hiçbir öğe ya da öznitelik içermeyen geçerli HTML5'tir. Bu sürümün asıl iddiası da bu ikisidir.

[Golfstack](https://www.fidelite.art/) projesinin bir parçasıdır.

## Oyna

- [cuneytinann.github.io/chessarbiter2kb](https://cuneytinann.github.io/chessarbiter2kb/)
- [fidelite.art/special/outofLevels/L2_string_flip_noBlockedDetector.html](https://www.fidelite.art/special/outofLevels/L2_string_flip_noBlockedDetector.html) — aynı dosya, proje sitesinde `L2` sürümleri arasında yansıtılmış hâli

Hedeflenen sınır 2.048 bayt; bu sürüm onun 1 bayt altında kalıyor.

Bir taşa tıklayın, sonra hedef kareye tıklayın. Yasal hedefler kehribar rengine döner, son hamle yeşil bir çerçeveyle işaretli kalır ve her yarım hamleden (ply) sonra tahta sırası gelen tarafa döner.

---

## Ne için var

Bu sürüm bir bayt rekoru kırmak için değil, bir sunum slaytında okunmak için var. Üç iddiada bulunuyor ve her biri kaynak kodda doğrulayabileceğiniz bir sayı.

**1. Herkesin atladığı kurallar aslında en ucuz olanlardır.** Kod golfü koleksiyonlarındaki satranç programları, yarım hamle sayacını, tekrar sayacını ve yetersiz materyal testini genellikle çıkarır; gerekçe, bunların kayıt tutmaktan ibaret olduğu ve kayıt tutmanın pahalıya patladığıdır. Oysa pahalı değiller:

| kural | uygulamanın tamamı | bayt |
| --- | --- | --- |
| Yarım hamle sayacı (`M=`) | `n=P\|b[f]>_?0:n+1` | **16** |
| Tekrar sayacı (`R=`) | `$=R[s=b+t+e+c]=-~R[s]` | **21** |
| Rok hakları, dört bitin tamamı | `C=i=>'20003001'[i%56]<<i/28` | **27** |
| Yetersiz materyal, iki taraf için aynı anda | `(m=W=0,b.map((p,i)=>p>_&&(q=j(p),q<'C'?m\|=(i/8^i)%2+1:W+=q>'N'?9:q=='N')),W*2+m<3)` | **82** |
| Oyunun bitebileceği her durum | `?'IM':$>4?'5R':n>149&&'75':l(t)?t?'B#':'W#':'SM'` | **48** |

Beş kural, 194 bayt, dosyanın onda biri. Bir satranç programının pahalı kısmı hiçbir zaman kural kitabı olmadı.

**2. Okunabilir bir tahta neredeyse bedavadır.** Dizi, sayısal taş kodları yerine bir FEN'in içerdiği harflerin aynısını tutar — `rnbqkbnr`, `P`, `-`. Aynı motorun sayısal sürümüyle, fonksiyon fonksiyon ve iki tarafta da aynı kural setiyle ölçüldüğünde, harflerin maliyeti **74 bayttır**: durum ve altı kural fonksiyonu genelinde 805'e karşı 880, dosyanın %4'ünden az. Karşılığında hata ayıklayıcıda doğrudan okunabilen bir tahta ve temsil ettiği pozisyona benzeyen bir tekrar anahtarı elde edilir.

**3. Yalnızca kendiliğinden devreye giren kurallar.** *Talep edilmesi* gereken bir beraberlik, onu talep edecek bir oyuncuya ihtiyaç duyar; bu bir düğme gerektirir, düğme de bir etkileşim katmanı gerektirir. Bu sürümde böyle bir katman yok, bu yüzden yalnızca hakemin kendiliğinden ilan ettiği bitiş durumlarını içerir: 5 kez tekrar, 75 hamle kuralı, yetersiz materyal, mat, pat. Sayaçlar yine de çalışır ve ekranda gösterilir — `R=` değerinin 3'e çıktığını ve `M=` değerinin 99'u geçtiğini izleyebilirsiniz — yalnızca burada oyunu bitirmezler, çünkü FIDE kurallarına göre bu iki eşik hakemin ilanıyla değil, oyuncunun talebiyle devreye girer.

---

## Tahta bir FEN kaydıdır

```js
b=[...'rnbqkbnrpppppppp'+_.repeat(32)+'PPPPPPPPRNBQKBNR'],t=1,c=15,e=_,n=0,
```

Bu tek satır, alan alan ve FEN'in kendi sırasıyla bir FEN'dir: **tahta, sırası gelen taraf, rok hakları, geçerken alma karesi, yarım hamle sayacı.** 0. indeks a8, 63. indeks h1'dir; bu hem bir FEN'in yazıldığı hem de bir tahtanın okunduğu sıradır. Büyük harf Beyaz, küçük harf siyah, `-` boş karedir.

Eksik olan tek FEN alanı tam hamle numarasıdır; eksiktir çünkü hiçbir kural onu okumaz. 50 ve 75 hamle kuralları yarım hamleleri sayar; tekrar ise hamle numarasıyla değil, bir pozisyon anahtarıyla belirlenir.

Bu gösterimin motorun her köşesinde sağladığı kolaylıklar:

```js
b[i]              kare                   tek dizi erişimi
j(b[i])           taş türü               'P', 'N', 'K' …
b[i]<E            renk                   büyük harf Beyaz'dır
b[i]==_           boş
b+t+e+c           tekrar anahtarı        tahta, taraf, ep karesi, haklar
```

Son satır ikinci kez bakmaya değer: FIDE'nin iki pozisyonun aynı olup olmadığına karar vermek için kullandığı anahtar, dört değişkenin dize birleştirmesidir ve yazdırdığınızda neredeyse bir FEN gibi okunur.

---

## Anatomi

Dosyanın her baytı, bölümlere göre.

| bölüm | bayt | |
| --- | --- | --- |
| işaretleme ve CSS | 252 | tahta, durum satırı, renkler, yerleşim |
| takma adlar | 59 | `_` `N` `E` `a` `j` |
| durum | 75 | yukarıdaki FEN satırı |
| `z`, `R` | 22 | sonuç kodu, tekrar tablosu |
| `G` | 293 | geometri: bu taş şu kareye ulaşabilir mi |
| `V` | 45 | bu kare saldırı altında mı |
| `l` | 30 | bu şah şah altında mı |
| `L` | 95 | bu hamle yasal mı — oyna, sor, geri al |
| `C` | 28 | bir karenin hangi rok hakkını düşürdüğü |
| `M` | 233 | hamleyi yap: sayaç, terfi, ep kurbanı, kale sıçraması, ep karesi |
| kurulum | 163 | terfi düğmeleri ve 64 kare, üretilmiş |
| `d` | 274 | tahtayı ve durum satırını çiz |
| `A` | 257 | hamleyi uygula, ardından karar |
| `S` | 95 | tıklama işleyicisi |
| BOM, `<script>` etiketleri, satır sonları | 126 | aşağıdaki yerleşim bu maliyete değer |

Başka bir şekilde bölersek: **928 bayt satranç, 1.119 bayt geri kalan her şey.** Kuralların maliyeti, onları gösteren tahtanın maliyetiyle hemen hemen aynı.

---

## İçinde neler var

- **Tüm taş hareketleri**, aritmetikten türetilmiş. Yön tablosu yok, ofset dizisi yok. At `h*v==2`'dir; uzun menzilli taşlar kendi tür harflerini bir yetenek testi olarak kullanır.
- **Tam yasallık kontrolü.** Kendi şahınızı şah altında bırakan bir hamle asla kabul edilmez. Her aday hamle kopyalanmış bir tahtada oynanır ve şahın durumu sorgulanır.
- **Rok**, iki yöne de, tüm koşullarıyla: hak hâlâ duruyor, kale yolu açık, şah şah altında değil, şah saldırı altındaki bir kareden geçmiyor, şah şah altına girmiyor.
- **Geçerken alma (en passant)**, neredeyse herkesin yanlış yaptığı kısım dahil — aşağıya bakın.
- **Seçicili terfi.** Vezir, kale, fil, at. Siz seçene kadar tahta kilitlenir ve hamle o ana kadar tamamlanmaz.
- **Şah, şah mat, pat**, birbirinden ayırt edilir.
- **Yetersiz materyal**, FIDE yorumuna göre: yalnız şahlar, tek at, tek fil ve hepsi aynı renk karede duran filler ölü pozisyondur; zıt renkli filler, iki at ve fil + at değildir.
- **5 kez tekrar** ve **75 hamle kuralı**, talep gerekmeden ilan edilir.
- Her yarım hamleden sonra sırası gelen tarafa **tahta çevirme**.
- **Durum satırı**: köşe koordinatları, sırası gelen taraf şah altındaysa `C!`, `M=` yarım hamle sayacı, `R=` mevcut pozisyonun kaç kez görüldüğü. Oyun bittiğinde satır sonuç koduna dönüşür.

## İçinde neler yok

- Saat yok.
- Beraberlik teklifi ve beraberlik talebi yok; dolayısıyla 3 kez tekrar talebi ve 50 hamle talebi de yok. Sayaçlar çalışır ve gösterilir; oyunu yalnızca otomatik eşikler bitirir.
- **Kilitli pozisyon** tespiti yok — FIDE 5.2.2'nin diğer yarısı; materyal yeterli olsa da piyonlar kilitlendiği için mat zaten imkânsızdır. Bu dedektör 497 bayt, yani bu boyuttaki bir dosyanın dörtte biri; bilerek çıkarıldı. Bu yüzden buradaki sonuç kodu `DP` (ölü pozisyon) değil `IM`'dir (yetersiz materyal), çünkü kod gerçekte neyin test edildiğini adlandırır. Kilitli bir pozisyon yine beraberlikle biter, sadece daha geç: tekrar ya da 75 hamle kuralı yoluyla.
- Bot yok, geri alma yok, FEN içe/dışa aktarma yok, PGN yok.

Eksiksiz FIDE hakemi için — saat, beraberlik teklifleri ve talepleri, terk, süre bitimi, ölü pozisyonlar, on beş sonuç kodu — [fidelite.art](https://www.fidelite.art/) adresine bakın.

## Sonuç kodları

Altı son, her biri iki karakter.

| kod | anlamı |
| --- | --- |
| `W#` | Beyaz mat eder |
| `B#` | Siyah mat eder |
| `SM` | Pat |
| `IM` | Yetersiz materyal |
| `5R` | 5 kez tekrar |
| `75` | 75 hamle kuralı |

---

## Kaynağı okumak

Dosya, bağımlılık sırasına dizilmiş, virgülle ayrılmış tek bir bildirim zinciridir. Her fonksiyon yalnızca kendinden öncekilere dayanır — ve her satırda tek bir kural olacak şekilde 56 satıra bölünmüştür. Dört taş ailesi art arda dört satırda durur. Bir hamle, bu beş adımın gerçekleşmesi gereken sırayla beş satırda yazılır. Oyunun bitebileceği tüm yollar dört satırlık bir basamak dizisidir. Bunların hiçbiri boşluk karakterlerinden fazlasına mal olmaz: tüm satır sonlarını silin, dosya bayt bayt eskisiyle aynı olur.

```
takma adlar  →  durum  →  G  →  V  →  l  →  L  →  C  →  M  →  ana kod
                geometri ── saldırı ── yasallık ── uygulama
```

`G`, her şeyi göz ardı ederek bir taşın bir kareye ulaşıp ulaşamayacağını sorar. `V`, bütün tahtaya bir karenin saldırı altında olup olmadığını sorar. `l`, bunu bir şah için sorar. `L`, yasallığı çözülebileceği tek şekilde çözer — hamleyi kopyalanmış bir tahtada oynar, `l`'ye sorar ve tahtayı geri koyar. `M` bir hamleyi yazar ve başka hiçbir şey yapmaz, çünkü `L` onu deneme olarak çağırır. Kalıcı bir şey yapan tek fonksiyon `A`'dır.

Bağımlılığın geriye doğru aktığı iki yer var ve ikisi de koddan değil, satranç kurallarından kaynaklanır. `G`, `V`'yi çağırır; çünkü şahın geçtiği karenin saldırı altında olup olmadığı sorulmadan rok değerlendirilemez. `M`, `L`'yi çağırır; çünkü geçerken alma karesi yalnızca alma gerçekten yasalsa yazılabilir.

### Seçme satırlar

```js
d=h|v                      hamlenin kaç kare kapsadığı, dallanma olmadan
k=(f-i)/d                  adım vektörü: ±1, ±8, ±7, ±9, tek bir bölmeyle
h*v==2                     at: 2 veren tek tamsayı çiftleri 1×2 ve 2×1'dir
y%5==1                     iki piyon başlangıç sırası — mod 5'te kalanı 1 olan tek sıralar
f%56<8                     son sıra, iki renk için de
f^8                        geçerken alma kurbanı, yönü XOR'un kendisiyle bulunur
i+3.5*k-.5                 rok kalesinin karesi, şah ve vezir kanadı tek ifadede
h==v&P<'R'                 çapraz gidenler: fil ve vezir geçer, kale elenir, alfabeye göre
P>'B'                      düz gidenler: kale ve vezir geçer, fil elenir
W*2+m<3                    yetersiz materyal tablosunun tamamı, tek karşılaştırma
-~R[s]                     ilk kez görülen pozisyon 1 değerini alır, varsayılan atama yok
'-BKNPQR'.search(j(b[u]))  harften taş simgesine, map yok, switch yok
```

Bunların en şıkı uzun menzilli taşlarla ilgili olanlardır. Alfabede `B < Q < R`, yani `P<'R'` tam olarak "çapraz gidebilir", `P>'B'` ise tam olarak "düz gidebilir" demektir. Vezir için ayrı bir dal yazmaya gerek yoktur; o, iki testi de geçen harften ibarettir.

### Geçerken almanın inceliği

Bu, tüm kural kitabındaki en öğretici hatadır ve `M`'nin yasallık katmanına geri uzanmasının nedenidir:

```js
e=P&d>9&&(e=q,[f-1,f+1].some(x=>b[x]=='Pp'[+g]&&L(x,e)))?e:_
```

Bir piyon az önce iki kare ilerledi. Basit bir uygulama geçerken alma karesini hemen yazar. Bu uygulama ise onu geçici olarak yazar, sonra komşu bir rakip piyonun bu almayı *gerçekten ve yasal olarak* yapıp yapamayacağını kontrol eder — yapamıyorsa `-` değerine geri döner.

Fark, almanın kendisinde hiçbir zaman ortaya çıkmaz: alma yasal değilse hamle üreteci onu her iki durumda da reddeder. Fark **tekrar sayacında** ortaya çıkar, çünkü `e` pozisyon anahtarının bir parçasıdır. FIDE iki pozisyonu ancak ikisinde de aynı hamleler mümkünse — geçerken alma dahil — aynı sayar. Kimsenin kullanamayacağı bir kare yazarsanız aynı pozisyon iki farklı anahtar altına düşer ve tekrar beraberliği geç devreye girer ya da hiç girmez. Berabere bitmesi gereken bir oyun bu yüzden kaybedilebilir.

Klasik örnek: Siyah g7–g5 oynar ve f5'teki beyaz piyon, f8'deki bir kale tarafından şahına karşı açmaza alınmıştır. `fxg6` hattı açıp şahı açıkta bırakacağından alma yasal değildir, dolayısıyla kare hiç yazılmaz. Doğru çalışan bir sayaçla yalnızca doğruymuş gibi görünen bir sayaç arasındaki fark, bu 60 bayttır.

---

## Hile değil, standart

Bayt golfü yapılmış HTML genellikle eskimiş işaretlemeyle çalışır, çünkü eskimiş işaretleme daha kısadır: `<center>`, `bgcolor`, hücrelerde `align` ve `width`, `cellspacing`, quirks mode. Bu sürüm bunların hiçbirini yapmaz. Her birinin maliyeti ölçüldü, ardından hepsi CSS'e taşındı.

Dosya **sıfır hata ve sıfır uyarıyla doğrulanır** — iddia edilmedi, W3C Nu doğrulayıcısıyla kontrol edildi. Yalnızca dosya da değil: tahta, terfi seçici ve 64 karenin tamamı çalışma zamanında üretildiği için, sayfanın gerçekte oluşturduğu DOM'un bir anlık görüntüsü ayrı bir belge olarak doğrulandı. İkisi de temiz çıktı.

| | |
| --- | --- |
| eskimiş öğeler | yok |
| eskimiş öznitelikler | yok |
| quirks mode | hayır — standart mod bilerek korunur |
| yukarıdakilerin tamamının maliyeti | eski işaretlemeyle yazılmış aynı sürüme karşı **45 bayt** |

Kural dışı gibi görünen bazı şeyler aslında tamamen kurallara uygundur. `</p>`, `</td>`, `</tr>` ve `</html>` atlanır; `<html>`, `<head>` ve `<body>` hiçbir zaman öğe olarak yazılmaz — bunların hepsi geçerli HTML'dir, kapanış etiketleri spesifikasyona göre isteğe bağlıdır. `</button>` **yazılır**, çünkü o isteğe bağlı değildir. Tırnaksız öznitelik değerleri de, değer boşluk veya tırnak içermediği sürece geçerlidir.

Kodlama bir `<meta charset>` öğesiyle değil, bir **UTF-8 BOM** ile bildirilir: yirmi bayta karşı üç bayt; ayrıca BOM, HTML kodlama algılama algoritmasının hem meta öğesinden hem de HTTP başlığından önce gelen ilk adımıdır. Doğrulayıcı onu bildirim olarak kabul eder. Bu dosyayı düzenlerseniz editörünüzün BOM'u koruduğundan emin olun, yoksa taş karakterleri bozulur.

Standart modu tercih etmek de bir heves meselesi değildir. Quirks mode'da `width` hücrenin iç boşluğunu da kapsar, kareler 66 genişlik ve 68 yükseklikte çıkar ve tahta kare olmaktan çıkar. Doctype 15 bayta mal olur ve tahtayı kare tutar.

Taş karakterleri Unicode `U+265A`–`U+265F` aralığındadır ve Beyaz için CSS ile yeniden renklendirilir. Görsel yok, yazı tipi indirme yok, CDN isteği yok.

---

## Doğrulama

Motor, okunarak değil çalıştırılarak kontrol edildi.

- Başlangıç pozisyonundan **perft**: 1–3 derinliklerinde 20 / 400 / 8.902. Kiwipete (CPW pozisyon 2): 48 / 2.039. CPW pozisyon 3: 14 / 191 / 2.812.
- **Geçerken alma**, dört durumun tamamı: açmazdaki alan piyon (kare yazılmaz), yasal alan piyon (yazılır ve tekrar anahtarında yer alır), alan piyon yok, alan piyon yanlış hatta.
- **Her sonuç kodu** gerçek bir pozisyondan yeniden üretildi — mat, pat, her materyal kombinasyonuyla yetersiz materyal, 16 yarım hamlede 5 kez tekrar, `M=150`'de 75 hamle kuralı.
- **Görüntüleme**, tam bir oyun boyunca, her seçimde ve her yarım hamlede önceki sürümle kare kare karşılaştırıldı — karakter, renk, arka plan, çerçeve, durum satırı.
- Sonuna kadar **rastgele kendi kendine oyun**, defalarca; her yarım hamleden sonra tahta dizisinin bozulup bozulmadığı kontrol edildi.
- **İşaretleme**, W3C Nu doğrulayıcısıyla doğrulandı; hem kaynak dosya hem de çalışma zamanı DOM'unun anlık görüntüsü.

---

## İlgili

- [chess1023byte](https://github.com/cuneytinann/chess1023byte) — ölçeğin öbür ucu: yalnızca temel kurallar, paketlenmiş, 1.023 baytta
- [fidelite.art](https://www.fidelite.art/) — eksiksiz hakem: saat, beraberlik teklifleri ve talepleri, terk, süre bitimi, ölü pozisyonlar, on beş sonuç kodu, sekiz ön yüz
## Lisans

MIT
