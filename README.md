Kalkulator Desktop - warna cream
Fitur : + - x / , akar, pangkat, per (1/x), modulo, ( ) [ ] { }, sin, cos, tan, pi
        Mode DEG/RAD, bisa diperkecil, diperbesar (maximize) dan ditutup.
Jalankan: python kalkulator.py
"""
import ast
import math
import operator
import re
import tkinter as tk
import tkinter.font as tkfont

# ---------- Warna (tema cream) ----------
CREAM = "#FFF3D6"        # latar utama
CREAM_TERANG = "#FFFBEF"  # tombol angka
CREAM_GELAP = "#F2DDAE"   # tombol operator
FUNGSI = "#EBD3A0"        # tombol fungsi ilmiah
AKSEN = "#D9A441"         # tombol sama dengan
HAPUS = "#E9B8A0"         # tombol hapus
TEKS = "#4A3B22"          # warna teks coklat tua
ERROR = "#B23A2E"

# ---------- Mesin hitung (aman, tanpa eval) ----------
class HitungError(Exception):
    pass


BIN_OPS = {
    ast.Add: operator.add,
    ast.Sub: operator.sub,
    ast.Mult: operator.mul,
    ast.Div: operator.truediv,
    ast.Pow: operator.pow,
    ast.Mod: operator.mod,
}
UN_OPS = {ast.UAdd: operator.pos, ast.USub: operator.neg}


def cek_kurung(teks):
    """Pastikan kurung ( ) [ ] { } berpasangan dengan benar."""
    pasangan = {")": "(", "]": "[", "}": "{"}
    tumpukan = []
    for ch in teks:
        if ch in "([{":
            tumpukan.append(ch)
        elif ch in ")]}":
            if not tumpukan or tumpukan.pop() != pasangan[ch]:
                raise HitungError("Kurung tidak cocok")
    if tumpukan:
        raise HitungError("Kurung belum ditutup")


def siapkan(ekspresi, derajat):
    s = ekspresi.strip()
    if not s:
        raise HitungError("Kosong")
    cek_kurung(s)
    s = (s.replace("×", "*").replace("÷", "/").replace("−", "-")
          .replace("^", "**").replace(",", ".").replace("π", "pi")
          .replace("√", "sqrt"))
    for k in "[{":
        s = s.replace(k, "(")
    for k in "]}":
        s = s.replace(k, ")")
    # perkalian implisit: 2(3), 2sin(30), (1)(2), 2pi, )3
    s = re.sub(r"(\d|\)|pi)\s*(?=\(|sin|cos|tan|sqrt|pi)", r"\1*", s)
    s = re.sub(r"(\)|pi)\s*(?=\d)", r"\1*", s)
    return s


def buat_fungsi(derajat):
    def sudut(x):
        return math.radians(x) if derajat else x

    def sin(x):
        return math.sin(sudut(x))

    def cos(x):
        return math.cos(sudut(x))

    def tan(x):
        if derajat and (x - 90) % 180 == 0:
            raise HitungError("tan tidak terdefinisi")
        return math.tan(sudut(x))

    def sqrt(x):
        if x < 0:
            raise HitungError("Akar bilangan negatif")
        return math.sqrt(x)

    return {"sin": sin, "cos": cos, "tan": tan, "sqrt": sqrt}


def evaluasi(node, fungsi):
    if isinstance(node, ast.Expression):
        return evaluasi(node.body, fungsi)
    if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
        return node.value
    if isinstance(node, ast.Name) and node.id == "pi":
        return math.pi
    if isinstance(node, ast.BinOp) and type(node.op) in BIN_OPS:
        kiri = evaluasi(node.left, fungsi)
        kanan = evaluasi(node.right, fungsi)
        if isinstance(node.op, ast.Pow) and abs(kanan) > 10000:
            raise HitungError("Pangkat terlalu besar")
        return BIN_OPS[type(node.op)](kiri, kanan)
    if isinstance(node, ast.UnaryOp) and type(node.op) in UN_OPS:
        return UN_OPS[type(node.op)](evaluasi(node.operand, fungsi))
    if (isinstance(node, ast.Call) and isinstance(node.func, ast.Name)
            and node.func.id in fungsi and len(node.args) == 1):
        return fungsi[node.func.id](evaluasi(node.args[0], fungsi))
    raise HitungError("Ekspresi tidak valid")


def hitung(ekspresi, derajat=True):
    s = siapkan(ekspresi, derajat)
    try:
        pohon = ast.parse(s, mode="eval")
        hasil = evaluasi(pohon, buat_fungsi(derajat))
    except HitungError:
        raise
    except ZeroDivisionError:
        raise HitungError("Tidak bisa dibagi nol")
    except OverflowError:
        raise HitungError("Angka terlalu besar")
    except (SyntaxError, ValueError, TypeError):
        raise HitungError("Ekspresi tidak valid")
    return format_hasil(hasil)


def format_hasil(x):
    if isinstance(x, complex):
        raise HitungError("Hasil kompleks")
    if isinstance(x, int):
        return str(x)
    x = round(x, 12)
    if x == int(x) and abs(x) < 1e15:
        return str(int(x))
    return format(x, ".12g")


# ---------- Tampilan ----------
class Kalkulator:
    def __init__(self, root):
        self.root = root
        self.derajat = True

        root.title("Kalkulator Krem")
        root.geometry("480x660")
        root.minsize(360, 500)
        root.configure(bg=CREAM)
        root.resizable(True, True)  # bisa diperkecil / diperbesar / ditutup

        self.f_ekspresi = tkfont.Font(family="Segoe UI", size=12)
        self.f_layar = tkfont.Font(family="Segoe UI", size=26, weight="bold")
        self.f_tombol = tkfont.Font(family="Segoe UI", size=13, weight="bold")

        root.columnconfigure(0, weight=1)
        root.rowconfigure(0, weight=2)
        root.rowconfigure(1, weight=3)
        root.rowconfigure(2, weight=4)

        self.buat_layar()
        self.buat_tombol()

        root.bind("<Configure>", self.ubah_ukuran)
        self.layar.focus_set()

    # --- layar ---
    def buat_layar(self):
        frame = tk.Frame(self.root, bg=CREAM)
        frame.grid(row=0, column=0, sticky="nsew", padx=10, pady=(10, 4))
        frame.columnconfigure(0, weight=1)
        frame.rowconfigure(1, weight=1)

        self.label_info = tk.Label(frame, text="", anchor="e", bg=CREAM,
                                   fg=TEKS, font=self.f_ekspresi)
        self.label_info.grid(row=0, column=0, sticky="ew")

        self.layar = tk.Entry(frame, justify="right", bd=0, relief="flat",
                              bg=CREAM_TERANG, fg=TEKS, insertbackground=TEKS,
                              font=self.f_layar,
                              highlightthickness=2,
                              highlightbackground=CREAM_GELAP,
                              highlightcolor=AKSEN)
        self.layar.grid(row=1, column=0, sticky="nsew", ipady=8)
        self.layar.bind("<Return>", lambda e: (self.sama_dengan(), "break")[1])
        self.layar.bind("<KP_Enter>", lambda e: (self.sama_dengan(), "break")[1])
        self.layar.bind("<Escape>", lambda e: self.bersihkan())

    # --- tombol ---
    def buat_tombol(self):
        # bagian ilmiah: 6 kolom x 3 baris
        ilmiah = tk.Frame(self.root, bg=CREAM)
        ilmiah.grid(row=1, column=0, sticky="nsew", padx=7)
        daftar = [
            [("DEG", self.ganti_mode, FUNGSI), ("sin", lambda: self.masuk("sin("), FUNGSI),
             ("cos", lambda: self.masuk("cos("), FUNGSI), ("tan", lambda: self.masuk("tan("), FUNGSI),
             ("π", lambda: self.masuk("π"), FUNGSI), ("C", self.bersihkan, HAPUS)],
            [("(", lambda: self.masuk("("), CREAM_GELAP), (")", lambda: self.masuk(")"), CREAM_GELAP),
             ("[", lambda: self.masuk("["), CREAM_GELAP), ("]", lambda: self.masuk("]"), CREAM_GELAP),
             ("{", lambda: self.masuk("{"), CREAM_GELAP), ("}", lambda: self.masuk("}"), CREAM_GELAP)],
            [("√", lambda: self.masuk("√("), FUNGSI), ("x²", lambda: self.masuk("^2"), FUNGSI),
             ("xʸ", lambda: self.masuk("^"), FUNGSI), ("per", self.per, FUNGSI),
             ("⌫", self.hapus_satu, HAPUS), ("%", lambda: self.masuk("%"), FUNGSI)],
        ]
        self.tombol_mode = None
        for r, baris in enumerate(daftar):
            ilmiah.rowconfigure(r, weight=1)
            for c, (teks, cmd, warna) in enumerate(baris):
                ilmiah.columnconfigure(c, weight=1, uniform="sci")
                b = self.tombol(ilmiah, teks, cmd, warna, r, c)
                if teks == "DEG":
                    self.tombol_mode = b

        # bagian utama: 4 kolom x 4 baris
        utama = tk.Frame(self.root, bg=CREAM)
        utama.grid(row=2, column=0, sticky="nsew", padx=7, pady=(0, 8))
        susunan = [
            ["7", "8", "9", "÷"],
            ["4", "5", "6", "×"],
            ["1", "2", "3", "−"],
            ["0", ".", "=", "+"],
        ]
        for r, baris in enumerate(susunan):
            utama.rowconfigure(r, weight=1)
            for c, teks in enumerate(baris):
                utama.columnconfigure(c, weight=1, uniform="utama")
                if teks == "=":
                    warna, cmd = AKSEN, self.sama_dengan
                elif teks in "÷×−+":
                    warna, cmd = CREAM_GELAP, (lambda t=teks: self.masuk(t))
                else:
                    warna, cmd = CREAM_TERANG, (lambda t=teks: self.masuk(t))
                self.tombol(utama, teks, cmd, warna, r, c)

    def tombol(self, parent, teks, cmd, warna, r, c):
        b = tk.Button(parent, text=teks, command=cmd, bg=warna, fg=TEKS,
                      activebackground=AKSEN, activeforeground=TEKS,
                      font=self.f_tombol, relief="flat", bd=0,
                      cursor="hand2", takefocus=0)
        b.grid(row=r, column=c, sticky="nsew", padx=3, pady=3)
        return b

    # --- aksi ---
    def masuk(self, teks):
        self.layar.insert(self.layar.index(tk.INSERT), teks)
        self.layar.focus_set()
        self.label_info.config(text="", fg=TEKS)

    def bersihkan(self):
        self.layar.delete(0, tk.END)
        self.label_info.config(text="", fg=TEKS)
        self.layar.focus_set()

    def hapus_satu(self):
        i = self.layar.index(tk.INSERT)
        if i > 0:
            self.layar.delete(i - 1)
        self.layar.focus_set()

    def per(self):
        """Kebalikan (1 per x): membungkus isi layar menjadi 1/(...)"""
        isi = self.layar.get().strip()
        if not isi:
            return
        self.layar.delete(0, tk.END)
        self.layar.insert(0, f"1÷({isi})")
        self.layar.focus_set()

    def ganti_mode(self):
        self.derajat = not self.derajat
        self.tombol_mode.config(text="DEG" if self.derajat else "RAD")
        self.layar.focus_set()

    def sama_dengan(self):
        ekspresi = self.layar.get()
        try:
            hasil = hitung(ekspresi, self.derajat)
        except HitungError as e:
            self.label_info.config(text=f"⚠ {e}", fg=ERROR)
            return
        self.label_info.config(text=f"{ekspresi} =", fg=TEKS)
        self.layar.delete(0, tk.END)
        self.layar.insert(0, hasil)
        self.layar.icursor(tk.END)

    # --- ukuran font ikut ukuran jendela ---
    def ubah_ukuran(self, event):
        if event.widget is not self.root:
            return
        skala = min(event.width / 480, event.height / 660)
        self.f_tombol.configure(size=max(9, int(13 * skala)))
        self.f_layar.configure(size=max(14, int(26 * skala)))
        self.f_ekspresi.configure(size=max(9, int(12 * skala)))


if __name__ == "__main__":
    root = tk.Tk()
    Kalkulator(root)
    root.mainloop()
