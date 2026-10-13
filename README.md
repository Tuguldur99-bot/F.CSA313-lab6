# Lab06 - Category-Partition ба PICT

Нэр: Ц.Төгөлдөр Оюутны код: B232270050

## Хэрэгсэл: PICT эх кодоос бүтээсэн

PICT-ийг `~/tools/pict` дотор (репоос гадна) clone хийж cmake-ээр бүтээсэн. `results/pict-build.txt`-ийн агуулга:

```
pict commit ab76c25 2026-09-08
Pairwise Independent Combinatorial Testing

Usage: pict model [options]
```

## Хэсэг A: Category-Partition (гараар) - шилжүүлгийн функц

Тестлэгдэх функц: `POST /transfer { from, to, amount, currency }`. Амжилт `201 {"result":"OK"}`, оролтын алдаа `200 {"result":"ERROR_..."}`.

### Олон алдааны давуу эрэмбэ

Нэг хүсэлтэд хэд хэдэн алдаа зэрэг байвал **from → to → amount** дарааллаар шалгана гэж тогтоосон: `ERROR_NO_FROM` → `ERROR_FROZEN` → `ERROR_NO_TO` → `ERROR_SAME_ACCOUNT` → `ERROR_BAD_AMOUNT` → `ERROR_INSUFFICIENT` → `ERROR_LIMIT`. Үндэслэл: эхлээд оролцогч дансууд байгаа эсэх, идэвхтэй эсэхийг (from, дараа нь to) шалгана, учир нь дүнг үлдэгдэл/лимиттэй харьцуулахад from данс заавал хэрэгтэй; дүнгийн хувьд оролт өөрөө буруу эсэхийг (`amount <= 0`) дансны төлөвөөс хамаарсан шалгалтаас (үлдэгдэл, дараа нь лимит) өмнө шалгана.

### A1. Сонголт ба төлөөлөх утгууд

| Параметр / сонголт | Төлөөлөх утга | Тайлбар |
|---|---|---|
| **from** | `Valid` | байгаа, царцаагдаагүй данс |
| | `Frozen` | байгаа боловч царцаагдсан данс |
| | `Missing` | огт байхгүй дансны ID |
| **to** | `Valid` | байгаа данс, from-оос өөр |
| | `Missing` | байхгүй дансны ID |
| | `SameAsFrom` | from-той ижил ID |
| **amount** | `Normal` | 0 < amount, үлдэгдэл ба лимитийн дотор |
| | `ExactBalance` | amount яг дансны үлдэгдэлтэй тэнцүү (хязгаарын утга) |
| | `Zero` | amount = 0 |
| | `Negative` | amount < 0 |
| | `OverBalance` | үлдэгдлээс их, лимитийн дотор |
| | `OverLimit` | үлдэгдлийн дотор, өдрийн лимитээс их |
| | `OverOnlyAfterConversion` | тоон утгаараа үлдэгдлээс бага, харин хөрвүүлсний дараа үлдэгдлээс их |
| **currency** | `Same` | дансны валюттай ижил |
| | `Different` | дансны валютаас өөр, ханшаар хөрвүүлэгдэнэ |

**Далд орчны сонголт.** Үлдэгдэл, өдрийн лимитийн хэрэглээ, ханш нь хүсэлтийн талбар биш, системийн төлөв. Иймд тэдгээрийг тусдаа параметр болгохгүй, `amount`-ийн утгуудаар (`OverBalance`, `OverLimit`, `OverOnlyAfterConversion`) илэрхийлсэн: тест бүр өөрийн setup-аар үлдэгдэл, лимитийг тохируулж тухайн ангийг үүсгэнэ. Тусдаа сонголт болгож нэмсэн зүйл бол "хөрвүүлсний дараа" шалгах дүрэм (`OverOnlyAfterConversion`), учир нь тодорхойлолт үлдэгдэл, лимитийг хөрвүүлсний дараах дүнгээр шалгана гэж заасан.

### A2. Хязгаарлалтууд

| Утга | Тэмдэглэгээ | Үндэслэл |
|---|---|---|
| from = `Frozen` | **[ERROR]** | `ERROR_FROZEN` буцаах ёстой, өөр утгатай хослуулбал алдаа бусдыг нууна |
| from = `Missing` | **[ERROR]** | `ERROR_NO_FROM` |
| to = `Missing` | **[ERROR]** | `ERROR_NO_TO` |
| to = `SameAsFrom` | **[ERROR]** | `ERROR_SAME_ACCOUNT` |
| amount = `Zero`, `Negative` | **[ERROR]** | `amount <= 0` буюу `ERROR_BAD_AMOUNT` |
| amount = `OverBalance` | **[ERROR]** | `ERROR_INSUFFICIENT` |
| amount = `OverLimit` | **[ERROR]** | `ERROR_LIMIT` |
| amount = `OverOnlyAfterConversion` | **[ERROR]** + **[IF currency = Different]** | `ERROR_INSUFFICIENT` боловч зөвхөн валют өөр үед утгатай: ижил валютад "хөрвүүлсний дараа" гэж байхгүй |
| amount = `ExactBalance` | **[SINGLE]** | хязгаарын утга, нэг удаа шалгахад хангалттай (үлдэгдэл бүрэн дуусгах нь ердийн амжилттай шилжүүлэг) |
| from = `Missing` үед amount-ийн `ExactBalance`/`OverBalance`/`OverLimit`/`OverOnlyAfterConversion` | **[IF from != Missing]** | from данс байхгүй бол үлдэгдэл, лимит, хөрвүүлэлт байхгүй тул эдгээр дүн утгагүй |
| from = `Missing` үед to = `SameAsFrom` | **[IF from != Missing]** | from байхгүй бол "to = from" гэж харьцуулах зүйл байхгүй |

### "N -> M" тооцоо

- Бүрэн олонлог: 3 (from) x 3 (to) x 7 (amount) x 2 (currency) = **126**
- Хязгаарлалтын дараа: [ERROR] утга бүр өөрөө ганц тест болно (бусад параметр хүчинтэй төлөөлөгчтэй): from 2 + to 2 + amount 4 (`Zero`, `Negative`, `OverBalance`, `OverLimit`) + `OverOnlyAfterConversion` 1 (currency зөвхөн `Different`) = **9**; хүчинтэй хослол: from=Valid, to=Valid, amount=Normal, currency 2 утга = **2**; [SINGLE] `ExactBalance` = **1**.
- Дүн: 9 + 2 + 1 = 12, өөрөөр хэлбэл **126 -> 12**.

### A3. Спецификацийн хүснэгт (15 спецификаци)

Дүн, үлдэгдлийн жишээ утгууд B хэсгийн тест кейст тодорхой өгөгдсөн.

| № | from | to | amount | currency | Хүлээгдэх үр дүн |
|---|---|---|---|---|---|
| S01 | Valid | Valid | Normal | Same | `201 OK` (happy path) |
| S02 | Valid | Valid | Normal | Different | `201 OK` (ханшийн хөрвүүлэлттэй) |
| S03 | Valid | Valid | ExactBalance [SINGLE] | Same | `201 OK` (үлдэгдэл 0 болно) |
| S04 | Missing | Valid | Normal | Same | `200 ERROR_NO_FROM` |
| S05 | Frozen | Valid | Normal | Same | `200 ERROR_FROZEN` |
| S06 | Valid | Missing | Normal | Same | `200 ERROR_NO_TO` |
| S07 | Valid | SameAsFrom | Normal | Same | `200 ERROR_SAME_ACCOUNT` |
| S08 | Valid | Valid | Zero | Same | `200 ERROR_BAD_AMOUNT` |
| S09 | Valid | Valid | Negative | Same | `200 ERROR_BAD_AMOUNT` |
| S10 | Valid | Valid | OverBalance | Same | `200 ERROR_INSUFFICIENT` |
| S11 | Valid | Valid | OverLimit | Same | `200 ERROR_LIMIT` |
| S12 | Valid | Valid | OverOnlyAfterConversion | Different | `200 ERROR_INSUFFICIENT` (хөрвүүлсний дараа үлдэгдлээс их) |
| S13 | Frozen | Missing | Normal | Same | `200 ERROR_FROZEN` - давхар алдаа, from нь to-оос өмнө шалгагдана |
| S14 | Valid | SameAsFrom | Zero | Same | `200 ERROR_SAME_ACCOUNT` - давхар алдаа, to нь amount-оос өмнө шалгагдана |
| S15 | Valid | Valid | үлдэгдэл БА лимит хоёулангаас их | Same | `200 ERROR_INSUFFICIENT` - давхар алдаа, үлдэгдлийн шалгалт лимитийнхээс өмнө |

S01-S12 нь дээрх "126 -> 12" тооцооны 12 тест (хүчинтэй 2 + [SINGLE] 1 + [ERROR] 9), S13-S15 нь давуу эрэмбийг шалгах нэмэлт давхар нөхцөлүүд.
