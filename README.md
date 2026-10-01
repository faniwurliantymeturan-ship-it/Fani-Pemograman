"""Kalkulator Krem: + - × ÷, akar, pangkat, per (1/x), modulo, ( ). Jalankan: python kalkulator.py"""
import ast, math, operator as op, re
import tkinter as tk

CREAM, TERANG, GELAP, FUNGSI, AKSEN, HAPUS = "#FFF3D6", "#FFFBEF", "#F2DDAE", "#EBD3A0", "#D9A441", "#E9B8A0"
TEKS, ERROR, FONT = "#4A3B22", "#B23A2E", "Segoe UI"
OPS = {ast.Add: op.add, ast.Sub: op.sub, ast.Mult: op.mul, ast.Div: op.truediv, ast.Pow: op.pow,
       ast.Mod: op.mod, ast.UAdd: op.pos, ast.USub: op.neg}
PESAN = {ZeroDivisionError: "Tidak bisa dibagi nol", OverflowError: "Angka terlalu besar"}

def evaluasi(n):
    if isinstance(n, ast.Constant) and isinstance(n.value, (int, float)):
        return n.value
    if isinstance(n, ast.BinOp) and type(n.op) in OPS:
        a, b = evaluasi(n.left), evaluasi(n.right)
        if isinstance(n.op, ast.Pow) and abs(b) > 10000:
            raise ValueError("Pangkat terlalu besar")
        return OPS[type(n.op)](a, b)
    if isinstance(n, ast.UnaryOp) and type(n.op) in OPS:
        return OPS[type(n.op)](evaluasi(n.operand))
    if isinstance(n, ast.Call) and getattr(n.func, "id", "") == "sqrt" and len(n.args) == 1:
        x = evaluasi(n.args[0])
        if x < 0:
            raise ValueError("Akar bilangan negatif")
        return math.sqrt(x)
    raise ValueError("Ekspresi tidak valid")

def hitung(teks):
    s = teks.strip()
    if not s:
        raise ValueError("Kosong")
    if s.count("(") != s.count(")"):
        raise ValueError("Kurung tidak cocok")
    for a, b in {"×": "*", "÷": "/", "−": "-", "^": "**", ",": ".", "√": "sqrt"}.items():
        s = s.replace(a, b)
    s = re.sub(r"(?<=[\d)])\s*(?=\(|sqrt)|(?<=\))\s*(?=\d)", "*", s)  # perkalian implisit: 2(3), )3
    try:
        h = evaluasi(ast.parse(s, mode="eval").body)
    except (ZeroDivisionError, OverflowError, SyntaxError, TypeError) as e:
        raise ValueError(PESAN.get(type(e), "Ekspresi tidak valid"))
    if isinstance(h, complex):
        raise ValueError("Hasil kompleks")
    h = round(h, 12)
    return str(int(h)) if h == int(h) and abs(h) < 1e15 else format(h, ".12g")

def bobot(w, baris, kolom):
    for i, b in enumerate(baris):
        w.rowconfigure(i, weight=b)
    for i, k in enumerate(kolom):
        w.columnconfigure(i, weight=k, uniform="k")

class Kalkulator:
    def __init__(self, root):
        root.title("Kalkulator Krem")
        root.geometry("400x560")
        root.minsize(300, 420)
        root.configure(bg=CREAM)
        bobot(root, (2, 7), (1,))
        atas = tk.Frame(root, bg=CREAM)
        atas.grid(row=0, column=0, sticky="nsew", padx=10, pady=(10, 4))
        bobot(atas, (0, 1), (1,))
        self.info = tk.Label(atas, anchor="e", bg=CREAM, fg=TEKS, font=(FONT, 12))
        self.info.grid(row=0, column=0, sticky="ew")
        self.layar = tk.Entry(atas, justify="right", bd=0, bg=TERANG, fg=TEKS, insertbackground=TEKS,
                              font=(FONT, 24, "bold"), highlightthickness=2,
                              highlightbackground=GELAP, highlightcolor=AKSEN)
        self.layar.grid(row=1, column=0, sticky="nsew", ipady=8)
        for k in ("<Return>", "<KP_Enter>"):
            self.layar.bind(k, lambda e: (self.sama_dengan(), "break")[1])
        self.layar.bind("<Escape>", lambda e: self.ganti(""))
        self.layar.focus_set()

        aksi = {"C": (lambda: self.ganti(""), HAPUS), "per": (self.per, FUNGSI), "=": (self.sama_dengan, AKSEN),
                "⌫": (lambda: self.layar.delete(self.layar.index(tk.INSERT) - 1, tk.INSERT), HAPUS)}
        sisip = {"√": "√(", "x²": "^2", "xʸ": "^"}
        baris = [["C", "(", ")", "⌫"], ["√", "x²", "xʸ", "per"], ["7", "8", "9", "÷"],
                 ["4", "5", "6", "×"], ["1", "2", "3", "−"], ["0", ".", "%", "+"], ["="]]
        grid = tk.Frame(root, bg=CREAM)
        grid.grid(row=1, column=0, sticky="nsew", padx=7, pady=(0, 8))
        bobot(grid, [1] * 7, [1] * 4)
        for r, isi in enumerate(baris):
            for c, t in enumerate(isi):
                warna = FUNGSI if t in sisip or t == "%" else GELAP if t in "()÷×−+" else TERANG
                cmd, warna = aksi.get(t, ((lambda x=sisip.get(t, t): self.masuk(x)), warna))
                tk.Button(grid, text=t, command=cmd, bg=warna, fg=TEKS, activebackground=AKSEN,
                          font=(FONT, 13, "bold"), relief="flat", bd=0, cursor="hand2", takefocus=0
                          ).grid(row=r, column=c, columnspan=4 if t == "=" else 1,
                                 sticky="nsew", padx=3, pady=3)

    def ganti(self, teks, info="", fg=TEKS):
        self.layar.delete(0, tk.END)
        self.layar.insert(0, teks)
        self.info.config(text=info, fg=fg)

    def masuk(self, t):
        self.layar.insert(self.layar.index(tk.INSERT), t)
        self.info.config(text="", fg=TEKS)

    def per(self):
        if self.layar.get().strip():
            self.ganti(f"1÷({self.layar.get().strip()})")

    def sama_dengan(self):
        e = self.layar.get()
        try:
            self.ganti(hitung(e), f"{e} =")
        except ValueError as x:
            self.info.config(text=f"⚠ {x}", fg=ERROR)

if __name__ == "__main__":
    root = tk.Tk()
    Kalkulator(root)
    root.mainloop()
