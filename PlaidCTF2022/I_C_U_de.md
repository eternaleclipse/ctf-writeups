# PlaidCTF 2022 I_C_U Writeup

__Intro__

**I_C_U** war eine spannende Herausforderung von [@jay_f0xtr0t](https://twitter.com/jay_f0xtr0t) und [@thebluepichu](https://twitter.com/thebluepichu) für PlaidCTF 2022. Sie wurde in die Kategorie "Misc" eingeordnet und umfasste zwei Flaggen, die mit 100 und 200 Punkten bewertet wurden.

Wenn man die Beschreibung der Herausforderung liest, geht es darum, Bilder nach Ähnlichkeit zu ordnen.

Wenn ich mir den Code (in Rust: Rust ist eine Multiparadigmen-Systemprogrammiersprache, die von der Open-Source-Community entwickelt wurde und unter anderem von Mozilla Research gesponsert wird. Sie wurde mit dem Ziel entwickelt, sicher, nebenläufig und praxisnah zu sein.) ansehe, fallen mir einige Dinge auf:
- Wir stellen zwei Dateien bereit, die einige Prüfungen bestehen müssen
- Einige JPEG/PNG-Header-Prüfungen
- Ein Texterkennungsalgorithmus - Tesseract 
- Ein [Perceptual hashing](https://en.wikipedia.org/wiki/Perceptual_hashing) Algorithmus - [img_hash](https://github.com/abonander/img_hash)

__Was ist also ein Wahrnehmungshash?__

Wie sich herausstellt, handelt es sich um eine Technologie, die in den letzten Jahren erforscht und entwickelt wurde.

Im Gegensatz zu kryptografischen Hashes (wie SHA, MD5, NTLM usw.), die so eindeutig wie möglich sein und Kollisionsangriffe verhindern sollen, sollen wahrnehmungsbezogene Hashes für Eingaben, die für den Menschen wahrnehmungsmäßig ähnlich sind, die gleichen oder ähnliche Ergebnisse liefern. 
Das bedeutet, dass der Algorithmus für zwei "ähnliche" Bilder den gleichen Hashwert liefern sollte.

Perceptual Hashes werden unter anderem bei der umgekehrten Bildsuche, der Erkennung von Videokopien, der Suche nach ähnlicher Musik und dem Abgleich von Gesichtern eingesetzt. 
Sie können als enge Verwandte von "Fuzzy-Hashes" (wie [ssDeep](https://ssdeep-project.github.io/)) betrachtet werden, die in der Forensik und bei der Erkennung/Klassifizierung von Malware stark genutzt werden.
 

__Setup__

Das mitgelieferte Archiv enthält praktischerweise ein `Dockerfile`(Docker ist eine Freie Software zur Isolierung von Anwendungen mit Hilfe von Containervirtualisierung. Docker vereinfacht die Bereitstellung von Anwendungen, weil sich Container, die alle nötigen Pakete enthalten, leicht als Dateien transportieren und installieren lassen), so dass ich nichts lokal auf meinem Rechner einrichten muss.

Beim anfänglichen Erstellen des Docker-Images gab es eine Fehlermeldung über ein fehlendes Modul, das die Flags enthalten sollte.
Anstatt das Modul zu erstellen, habe ich `main.rs` manuell mit einigen Flags gepatcht:

```rust
// mod secret;
// use crate::secret::{FLAG, FLAG_TOO};
...
  println!("Congratulations: {}", "flag1");
...
  println!("Wow, impressive: {}", "flag2");

```

Erstellen und Ausführen des Abbilds (mit einem Host [volume](https://docs.docker.com/storage/volumes/) für einfachen Dateizugriff vom Host):

```bash
docker build -t icu .
docker run --rm -it icu -v /opt/hostvol:/vol bash
```

Testen wir das Programm innerhalb des Containers mit den mitgelieferten Beispielbildern:
```sh
root@8f173908e9c0:/icu# ./target/release/i_c_u /vol/img1.png /vol/img2.png 
Image1 hash: ERsrE6nTHhI=
Image2 hash: AaXn5FkzdkA=
Hamming Distance: 31
Image 1 text: Sudo please
Image 2 text: give me the flag
Try again
```

Cool, dann fangen wir mal an! =) 

## Teil I - Myopie

Schauen wir uns den Code an und sehen wir uns an, was er mit unseren Bildern macht.

Wir liefern dem Programm zwei Dateien, entweder direkt oder über `stdin` mit Strings, die base64-kodierte Bilder enthalten.
Wenn base64 verwendet wird, darf die (kodierte) Länge 200.000 Bytes nicht überschreiten.
```rust
println!("Image 1 (base64):");
let mut s: String = String::new();
std::io::stdin().read_line(&mut s).unwrap();

if s.len() > 200_000 {
  println!("Too big");
  std::process::exit(2);
}
```

Nach dem Lesen der Dateien wird eine einfache Header-Prüfung durchgeführt, um sicherzustellen, dass es sich um PNG oder JPEG handelt.
```rust
let suffix = {
  if b1[0..4] == b2[0..4] && b1[0..4] == [0x89, 0x50, 0x4e, 0x47] {
    "png"
  } else if b1[6..10] == b2[6..10] && b1[6..10] == [0x4a, 0x46, 0x49, 0x46] {
    "jpg"
  } else {
    println!("Unknown formats");
    std::process::exit(3);
  }
};
```

Die Bilder werden mit dem Wahrnehmungshash verschlüsselt.
```rust
let hash1 = hasher.hash_image(&image1);
let hash2 = hasher.hash_image(&image2);

println!("Image1 hash: {}", hash1.to_base64());
println!("Image2 hash: {}", hash2.to_base64());
```

Das Programm berechnet die [Hamming-Distanz](https://en.wikipedia.org/wiki/Hamming_distance) zwischen den beiden Hashes - das ist einfach die Anzahl der Bits, die sich unterscheiden.
```rust
let dist = hash1.dist(&hash2);
println!("Hamming Distance: {}", dist);
```

Tesseract wird verwendet, um englischen Text aus den Bildern zu extrahieren. Das ist eine freie Software zur Texterkennung. Schwerpunkt ist die Erkennung von Textzeichen bzw. Textzeilen, aber auch die Zerlegung eines Textes in Textblöcke kann Tesseract übernehmen. Zur Verbesserung der Erkennungsraten verwendet Tesseract Sprachmodelle wie beispielsweise Wörterbücher.
```rust
let text1 = tesseract::ocr(&fn1, "eng")
        .unwrap()
        .trim()
        .split_whitespace()
        .collect::<Vec<_>>()
        .join(" ");
    let text2 = tesseract::ocr(&fn2, "eng")
        .unwrap()
        .trim()
        .split_whitespace()
        .collect::<Vec<_>>()
        .join(" ");

    println!("Image 1 text: {}", text1);
    println!("Image 2 text: {}", text2);
```

__Die Prüfungen__

Für jede Flagge gibt es einen anderen Satz von Prüfungen. Wir konzentrieren uns vorerst auf die erste Flagge.

```rust
if dist == 0
  && text1.to_lowercase() == "sudo please"
  && text2.to_lowercase() == "give me the flag"
  && hash1.to_base64() == "ERsrE6nTHhI="
{
  println!("Congratulations: {}", "flag1");
} else if ...
```

Um das Flag zu erhalten, brauchen wir also:
- Gültiger PNG- oder JPEG-Magic-Header am Anfang der Datei
- Die Hashes der Bilder sollten gleich sein (dist == 0 bedeutet, dass alle Bits gleich sind)
- Der erkannte Text (Groß- und Kleinschreibung wird nicht berücksichtigt: Der englische Ausdruck case sensitivity bezeichnet in der elektronischen Datenverarbeitung allgemein die Art und Weise, wie eine Rechenmaschine oder Programmiersprache die Unterscheidung von Groß- und Kleinschreibung handhabt) der ersten Datei muss "sudo please" lauten.
- Der erkannte Text (Groß- und Kleinschreibung wird nicht berücksichtigt) der zweiten Datei sollte "give me the flag" lauten.
- Der Hash für beide Bilder sollte "ERsrE6nTHhI=" lauten.

Moment, diesen Hash haben wir schon einmal gesehen! Als wir das Programm zum ersten Mal mit den Beispielen ausgeführt haben. Es ist der Hash für `img1.png`, der "sudo please" enthält.

Jetzt müssen wir nur noch ein Bild erstellen, das denselben Hash hat, aber tatsächlich einen Text enthält, der als "Gib mir die Flagge" erkannt wird.

__Angriffsansätze__

Als ich mir das Problem ansah, dachte ich zunächst an mehrere Möglichkeiten, es zu lösen:
- Den Integer `dist` irgendwie überlaufen lassen, so dass er `0` wird - Keine Option, der maximale Abstand wäre die Länge des Hash-Strings und der ist wirklich kurz. Außerdem schützt Rust vor int-Überläufen (Verwenden Sie den Wrapping-Typ, oder verwenden Sie die Wrapping-Funktionen direkt. Diese schalten die Überlaufprüfungen aus. Der Wrapping-Typ erlaubt es Ihnen, die normalen Operatoren wie gewohnt zu verwenden.) 
- Die Hamming-Distanz-Funktion irgendwie brechen - Es ist wirklich einfach, also sollte es keine ernsthaften Bugs geben, aber trotzdem
- Den Tesseract / `img_hash` Parser brechen - Das Dateiformat korrumpieren, so dass der `img_hash` Parser eine andere Datei liest als Tesseract. Vielleicht eine andere eingebettete Bilddatei verwenden oder die Header manipulieren
- In der Kryptographie wird bei einem Kollisionsangriff auf einen kryptographischen Hash versucht, zwei Eingaben zu finden, die denselben Hash-Wert ergeben, d.h. eine Hash-Kollision. Dies steht im Gegensatz zu einem Preimage-Angriff, bei dem ein bestimmter Ziel-Hashwert angegeben wird.. 
- Bruteforce für eine Hash-Kollision mit [Pillow](https://github.com/python-pillow/Pillow) - Da der Hash kurz ist, sollte es wahrscheinlich einfach sein, eine Kollision zu finden. Ich könnte nach dem Zufallsprinzip Sätze von generierten Bildern erstellen, die "Gib mir die Flagge" mit verschiedenen Schriftarten, Größen, Farben und Positionen enthalten, bis ich einen passenden Hash erhalte
- Lesen Sie den eigentlichen Code für `hash_img` und verstehen Sie, was er tut

 
__Spaß mit MSPaint__

Natürlich können Sie die Datei mit dem guten alten Microsoft Paint öffnen und damit spielen und sehen, wie sich der Hash ändert, wenn Sie das Bild verändern.

Das können Sie herausfinden:
- Kleine Änderungen haben keinen Einfluss auf den Hash
- Die Vergrößerung des Bildes wirkt sich nur geringfügig auf den Hash-Wert aus und lässt mehr Spielraum, um die OCR mit größerem Text zu füttern.
- Wenn man verschiedene Teile des Bildes ändert, ändert man auch verschiedene Teile des Hashwerts
- Das Spielen mit Farben kann den Wert verändern - es schien, als ob das Hinzufügen von Weiß den Wert für den spezifischen Teil des Hashes, der verändert wurde, erhöhte.

Bei der OCR ging es darum, den vorhandenen Text ("sudo please") so zu verformen, dass er ignoriert wird, und die richtige Positionierung, Schriftart und Größe für den neuen Text zu finden, damit er nur "Gib mir die Flagge" erkennt und keine zusätzlichen Kauderwelschzeichen hinzufügt. Als nächstes werden Sie von diesem Meisterwerk überrascht sein:

![gg](gmtf.jfif)

Beachten Sie die kleine Zeichenfolge in der oberen rechten Ecke - das ist das, was von der OCR gelesen wird. (Texterkennung ist ein Begriff aus der Informationstechnik. Es bezeichnet die automatisierte Texterkennung bzw. automatische Schrifterkennung innerhalb von Bildern. Ursprünglich basierte die automatische Texterkennung auf optischer Zeichenerkennung.) 

## Teil II - Hyperopie

Nach meinen Abenteuern mit Paint wollen wir uns nun die nächsten Flaggenprüfungen ansehen:

```rust
else if dist > 0
    && text1.to_lowercase() == "give me the flag"
    && text2.to_lowercase() == "give me the flag"
    && bindiff_of_1(&fn1, &fn2)
{
    println!("Wow, impressive: {}", "flag2");
}
```

__Prüft auf dieses Flag__:
- Der Header sollte PNG / JPEG sein. JPEG und PNG sind beide eine Art von Bildformat zum Speichern von Bildern. JPEG verwendet einen verlustbehafteten Komprimierungsalgorithmus und das Bild kann einen Teil seiner Daten verlieren, während PNG einen verlustfreien Komprimierungsalgorithmus verwendet und im PNG-Format kein Bilddatenverlust auftritt. LCA ist die verlustbehaftete Komprimierung oder irreversible Komprimierung die Klasse von Datenkomprimierungsverfahren, die ungenaue Annäherungen und teilweises Verwerfen von Daten verwendet, um den Inhalt darzustellen.
- Die Hashes sollten sich unterscheiden (um mindestens 1 Bit)
- Der für beide Bilder erkannte Text sollte "give me the flag" lauten
- Die gesamte Differenz zwischen den beiden Dateien sollte genau 1 Bit betragen

Die Implementierung für `bindiff_of_1` sieht solide aus. Sie prüft im Grunde, ob die Länge gleich ist, vergleicht die Inhalte der beiden Dateien mit XOR und prüft, ob die Summe 1 ist.

```rust
fn bindiff_of_1(fn1: &str, fn2: &str) -> bool {
    use std::io::Read;
    let b1: Vec<u8> = std::fs::File::open(fn1)
        .unwrap()
        .bytes()
        .collect::<Result<_, _>>()
        .unwrap();
    let b2: Vec<u8> = std::fs::File::open(fn2)
        .unwrap()
        .bytes()
        .collect::<Result<_, _>>()
        .unwrap();
    if b1.len() != b2.len() {
        return false;
    }
    b1.into_iter()
        .zip(b2.into_iter())
        .map(|(x, y)| (x ^ y).count_ones())
        .sum::<u32>()
        == 1
}
```

__Angriffsansätze__

Genau wie zuvor, sind dies einige Gedanken, wie man diesen Teil angehen kann:
- Überlauf der Summe
- Wenn Sie die Funktionen SUM, SUM_FLOAT und AVG mit einem NUMERIC-Datentyp verwenden, sollten Sie sich bewusst sein, dass ein Überlauf auftreten kann und wie Vertica auf diesen Überlauf reagiert.
- Überprüfen Sie den PNG-/JPEG-Header und suchen Sie nach einem Bit, das die Datei beschädigen könnte, so dass eine äquivalente, leere oder anderweitig fehlerhafte Datei entsteht, die für beide Bilder denselben Hash erzeugt.

Sie können das Bild herunterskalieren und ein schnelles und schmutziges Python-Skript schreiben, um jedes Bit im Bild zu spiegeln und auf Erfolg zu testen.


```python
import subprocess
import sys

def flip_bit(buf, i):
    byte_offset = i // 8
    bit_offset = i % 8
    byte = buf[byte_offset]
    byte ^= (1 << bit_offset)
    buf[byte_offset] = byte
    return buf


m1_contents = bytearray(open("/vol/m1.jpg", "rb").read())
m1_len = len(m1_contents)

for i in range(m1_len * 8):
    m2 = flip_bit(m1_contents[:], i)
    open("/vol/m2.jpg", "wb").write(m2)
    try:
        output = subprocess.check_output(["./target/release/i_c_u", "/vol/m1.jpg", "/vol/m2.jpg"])
        if b"Wow, impressive" in output:
            print(output)
            sys.exit(0)
    except subprocess.CalledProcessError:
        pass
```

Beim Schreiben und Testen dieses Skripts habe ich die [Docker-Erweiterung für VSCode] (https://code.visualstudio.com/docs/containers/overview) verwendet, was die Arbeit direkt im Container sehr erleichtert hat.

Nachdem ich das Skript gut 25 Minuten lang laufen ließ, fand es eine Position, an der das Umdrehen des Bits denselben Text und Hash ergab, und das Flag erschien.

## Senden der Ergebnisse und Abrufen der aktuellen Flaggen

Jetzt müssen wir unsere Bilder an den Server senden und unsere hart verdienten Flaggen abrufen!

Als ich zum ersten Mal versuchte, die Dateien im Terminal mit base64 zu kodieren und einzufügen, stürzte der Server mit einem Fehler ab - `InvalidLastSymbol(4094, 78)`. Wenn ich die [docs] (https://docs.rs/base64/0.10.0/base64/enum.DecodeError.html) lese, scheint es, dass das letzte base64 kodierte Zeichen (Symbol), das Rust erhalten hat, `N` war. Beim Versuch, es lokal in meinen Docker-Container einzufügen, stürzte es ebenfalls mit demselben Fehler ab.

Dies ließ mich sofort vermuten, dass eine Art von Copy-Paste-Trunkierung Hexerei vor sich ging. Tatsächlich funktionierte es, wenn ich die Dateien durch `stdin` leitete:

```bash
(base64 -w0 img1.png; echo; base64 -w0 img2.png) | ./i_c_u
```

Ich dachte daran, dasselbe mit `nc` zu machen, aber wenn man mit dem Server interagiert, gibt es einen Proof-of-Work-Befehl ([hashcash](https://en.wikipedia.org/wiki/Hashcash)), der lokal ausgeführt werden muss, bevor man auf die Challenge zugreifen kann. Ich habe zuerst versucht, meinen Befehl mit `Expect` zu verpacken, aber schließlich habe ich einfach ein Python-Skript geschrieben, das die ganze Sache erledigt. 
Base64 ist ein Verfahren zur Kodierung von 8-Bit-Binärdaten in eine Zeichenfolge, die nur aus lesbaren, Codepage-unabhängigen ASCII-Zeichen besteht. Es findet im Internet-Standard Multipurpose Internet Mail Extensions Anwendung und wird dort zum Versenden von E-Mail-Anhängen verwendet.

```python
from socket import *
import subprocess
import base64
import sys

def read_b64(filename):
    with open(filename, 'rb') as f:
        return base64.b64encode(f.read())

s = socket(AF_INET, SOCK_STREAM)
s.connect(("icu.chal.pwni.ng", 1337))
hashcash_cmd = s.recv(1024)

if not hashcash_cmd.startwith("hashcash "):
    sys.exit(1)

output = subprocess.check_output(hashcash_cmd, shell=True)
s.send(output)

print(s.recv(1024))
s.send(read_b64("1.png") + b"\n")

print(s.recv(1024))
s.send(read_b64("2.png") + b"\n")

print(s.recv(1024))
print(s.recv(1024))
print(s.recv(1024))
print(s.recv(1024))
```

Wir haben unsere Flaggen, toll!

## Abschließende Gedanken

Wir brauchen mehr Herausforderungen wie diese! Ein Aspekt dieser Herausforderung, der mir besonders gut gefallen hat, ist die Tatsache, dass es neben der Fehlersuche auch darum geht, mit einigen interessanten Technologien zu experimentieren. Der Aspekt des Rätseldesigns ist ebenfalls sehr gut - es handelt sich um ein interessantes Problem und nicht um einen superspezifischen Blödsinn, den sich jemand ausgedacht hat. Es gibt mehrere Möglichkeiten, an das Problem heranzugehen, und es gibt kein Rätselraten - man muss nur herausfinden, wie man zwei Algorithmen aus der realen Welt zum Laufen bringt.
