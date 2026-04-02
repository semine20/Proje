import sys, random, csv, os
from abc import ABC, abstractmethod
from PyQt5.QtWidgets import *
from PyQt5.QtCore import Qt
from PyQt5.QtGui import QIcon, QFont, QPixmap, QPalette, QBrush

# --- NYP TEMELLERİ (Mantık Kısmı) ---
class OzelYetenek(ABC):
    @abstractmethod
    def uygula(self, sporcu, temel_puan): pass
    def ad(self): return self.__class__.__name__

class ClutchPlayer(OzelYetenek):
    def uygula(self, sporcu, temel_puan): return temel_puan + 10 if sporcu.enerji < 40 else temel_puan

class Veteran(OzelYetenek):
    def uygula(self, sporcu, temel_puan): return temel_puan

class Legend(OzelYetenek):
    def uygula(self, sporcu, temel_puan): return temel_puan * 1.2

class Sporcu(ABC):
    def __init__(self, ad, takim, ozellikler, dayaniklilik, enerji, yetenek_adi):
        self.ad = ad
        self.takim = takim
        self.ozellikler = ozellikler
        self.dayaniklilik = int(dayaniklilik)
        self.enerji = int(enerji)
        self.seviye = 1
        self.deneyim = 0
        
        yetenekler = {"ClutchPlayer": ClutchPlayer(), "Veteran": Veteran(), "Legend": Legend()}
        self.yetenek = yetenekler.get(yetenek_adi, ClutchPlayer())

    @abstractmethod
    def brans(self): pass

    def performans_hesapla(self, oz_adi, moral):
        temel = self.ozellikler[oz_adi]
        ceza = 0
        if 40 <= self.enerji <= 70: ceza = temel * 0.1
        elif 0 < self.enerji < 40: ceza = temel * 0.2
        
        m_bonus = 10 if moral > 80 else (-5 if moral < 50 else 0)
        s_bonus = (self.seviye - 1) * 5
        
        sonuc = temel + m_bonus + s_bonus - ceza
        return self.yetenek.uygula(self, sonuc)

    def enerji_guncelle(self, durum):
        degisim = {"win": -5, "lose": -10, "draw": -3}[durum]
        if self.yetenek.ad() == "Veteran": degisim /= 2 
        self.enerji = max(0, self.enerji + degisim)

    def deneyim_guncelle(self, durum):
        self.deneyim += {"win": 2, "draw": 1, "lose": 0}[durum]
        if self.deneyim >= 4: self.seviye = 2
        if self.deneyim >= 8: self.seviye = 3

# --- HATANIN DÜZELTİLDİĞİ KISIM (Alt alta yazıldı) ---
class Futbolcu(Sporcu): 
    def brans(self): 
        return "futbol"

class Basketbolcu(Sporcu): 
    def brans(self): 
        return "basketbol"

class Voleybolcu(Sporcu): 
    def brans(self): 
        return "voleybol"

# -----------------------------------------------------

class OrtaStrateji:
    def sec(self, kartlar, brans):
        uygun = [k for k in kartlar if k.brans() == brans and k.enerji > 0]
        return max(uygun, key=lambda x: sum(x.ozellikler.values())) if uygun else None

class OyunSistemi:
    def __init__(self):
        self.kullanici_kartlar = []
        self.pc_kartlar = []
        self.skorlar = {"Sen": 0, "PC": 0}
        self.moral = 70
        self.tur_sirasi = ["futbol", "basketbol", "voleybol"] * 4
        self.aktif_tur = 0

    def dosya_oku(self):
        try:
            dosya_yolu = os.path.join(os.path.dirname(os.path.abspath(__file__)), "sporcular.csv")
            with open(dosya_yolu, "r", encoding="utf-8") as f:
                reader = csv.reader(f, delimiter=';')
                liste = []
                for r in reader:
                    if not r or len(r) < 9: continue
                    r = [i.strip() for i in r]
                    bilgi = {"P1": int(r[3]), "P2": int(r[4]), "P3": int(r[5])}
                    if r[0] == "futbol": k = Futbolcu(r[1], r[2], bilgi, r[6], r[7], r[8])
                    elif r[0] == "basketbol": k = Basketbolcu(r[1], r[2], bilgi, r[6], r[7], r[8])
                    else: k = Voleybolcu(r[1], r[2], bilgi, r[6], r[7], r[8])
                    liste.append(k)
            random.shuffle(liste)
            self.kullanici_kartlar = liste[:12]
            self.pc_kartlar = liste[12:]
            return True
        except: return False


# --- GÖRSEL KART TASARIMI ---
class SporcuKarti(QFrame):
    def __init__(self, sporcu, indeks, ana_pencere, aktif=True):
        super().__init__()
        self.indeks = indeks
        self.ana_pencere = ana_pencere
        self.setFixedSize(180, 260)
        self.setCursor(Qt.PointingHandCursor)
        self.aktif = aktif
        
        self.normal_stil = """
            QFrame { background-color: #ffffff; border-radius: 12px; border: 1px solid #dee2e6; }
            QFrame:hover { border: 2px solid #0d6efd; }
        """
        self.secili_stil = """
            QFrame { background-color: #e9ecef; border-radius: 12px; border: 3px solid #0d6efd; }
        """
        self.setStyleSheet(self.normal_stil)
        layout = QVBoxLayout(self)
        
        # --- AKILLI RESİM ARAMA (resimler klasörüne öncelik verir) ---
        dizin = os.path.dirname(os.path.abspath(__file__))
        ad_kucuk = sporcu.ad.lower()
        ad_alt_tire = ad_kucuk.replace(" ", "_")
        ad_noktasiz = ad_alt_tire.replace(".", "")
        
        olasi_yollar = [
            os.path.join(dizin, "resimler", f"{ad_alt_tire}.jpg"),
            os.path.join(dizin, "resimler", f"{ad_alt_tire}.jpeg"),
            os.path.join(dizin, "resimler", f"{ad_alt_tire}.png"),
            os.path.join(dizin, "resimler", f"{ad_noktasiz}.jpg"),
            os.path.join(dizin, "resimler", f"{ad_noktasiz}.jpeg"),
            os.path.join(dizin, "resimler", f"{ad_kucuk}.jpg")
        ]
        
        bulunan_resim = None
        for yol in olasi_yollar:
            if os.path.exists(yol):
                bulunan_resim = yol
                break
                
        self.img_lbl = QLabel()
        if bulunan_resim:
            pix = QPixmap(bulunan_resim).scaled(150, 150, Qt.KeepAspectRatioByExpanding, Qt.SmoothTransformation)
            self.img_lbl.setPixmap(pix)
        else:
            self.img_lbl.setText(f"Resmi Bulamadım\n({ad_alt_tire})")
            self.img_lbl.setAlignment(Qt.AlignCenter)
            self.img_lbl.setStyleSheet("color: gray;")
            
        self.img_lbl.setAlignment(Qt.AlignCenter)
        
        self.ad_lbl = QLabel(f"<b>{sporcu.ad}</b>")
        self.ad_lbl.setAlignment(Qt.AlignCenter)
        self.ad_lbl.setStyleSheet("color: #212529; font-size: 13px;")
        
        brans_lbl = QLabel(sporcu.brans().upper())
        brans_lbl.setAlignment(Qt.AlignCenter)
        brans_lbl.setStyleSheet("color: #6c757d; font-size: 10px;")
        
        self.bar = QProgressBar()
        self.bar.setValue(int(sporcu.enerji))
        self.bar.setFixedHeight(10)
        self.bar.setTextVisible(False)
        renk = "#198754" if sporcu.enerji > 40 else "#dc3545"
        self.bar.setStyleSheet(f"QProgressBar {{ border: 1px solid #ccc; border-radius: 5px; background: #f8f9fa; }} QProgressBar::chunk {{ background-color: {renk}; border-radius: 5px; }}")
        
        layout.addWidget(self.img_lbl)
        layout.addWidget(self.ad_lbl)
        layout.addWidget(brans_lbl)
        layout.addWidget(self.bar)
        
        if not aktif or sporcu.enerji <= 0:
            op = QGraphicsOpacityEffect(self)
            op.setOpacity(0.4)
            self.setGraphicsEffect(op)

    def mousePressEvent(self, event):
        if self.aktif: self.ana_pencere.kart_secildi(self.indeks)

    def secili_yap(self, durum):
        if durum: self.setStyleSheet(self.secili_stil)
        else: self.setStyleSheet(self.normal_stil)

# --- ANA EKRAN VE ARKA PLAN YÖNETİMİ ---
class OyunEkrani(QMainWindow):
    def __init__(self):
        super().__init__()
        self.sistem = OyunSistemi()
        self.ai = OrtaStrateji()
        self.secili_indeks = -1
        self.kart_widgetlari = []
        
        self.setWindowTitle("Sporcu Kart Ligi")
        self.setMinimumSize(1000, 700)
        
        dizin = os.path.dirname(os.path.abspath(__file__))
        bg_yol_jpg = os.path.join(dizin, "background.jpg")
        bg_yol_jpeg = os.path.join(dizin, "background.jpeg")
        bg_yol = bg_yol_jpg if os.path.exists(bg_yol_jpg) else bg_yol_jpeg
        
        if os.path.exists(bg_yol):
            pixmap = QPixmap(bg_yol)
            palette = QPalette()
            palette.setBrush(QPalette.Window, QBrush(pixmap.scaled(self.size(), Qt.KeepAspectRatioByExpanding, Qt.SmoothTransformation)))
            self.setPalette(palette)
            self.setAutoFillBackground(True)
        else:
            self.setStyleSheet("background-color: #f8f9fa;")

        self.ekranlar = QStackedWidget()
        self.ekranlar.setStyleSheet("background-color: transparent;") 
        self.setCentralWidget(self.ekranlar)
        
        self.giris_ekrani_kur()
        self.oyun_ekrani_kur()
        
        if self.sistem.dosya_oku(): 
            self.kartlari_ciz()

    def resizeEvent(self, event):
        dizin = os.path.dirname(os.path.abspath(__file__))
        bg_yol_jpg = os.path.join(dizin, "background.jpg")
        bg_yol_jpeg = os.path.join(dizin, "background.jpeg")
        bg_yol = bg_yol_jpg if os.path.exists(bg_yol_jpg) else bg_yol_jpeg
        
        if os.path.exists(bg_yol):
            pixmap = QPixmap(bg_yol)
            palette = QPalette()
            palette.setBrush(QPalette.Window, QBrush(pixmap.scaled(event.size(), Qt.KeepAspectRatioByExpanding, Qt.SmoothTransformation)))
            self.setPalette(palette)
        super().resizeEvent(event)

    def giris_ekrani_kur(self):
        widget = QWidget()
        widget.setStyleSheet("background-color: transparent;") 
        layout = QVBoxLayout(widget)
        layout.setAlignment(Qt.AlignCenter)
        
        baslik = QLabel("SPORCU KART OYUNU")
        baslik.setStyleSheet("font-size: 55px; font-weight: bold; color: #ffffff; background-color: rgba(0,0,0,150); padding: 10px; border-radius: 10px; margin-bottom: 10px;")
        baslik.setAlignment(Qt.AlignCenter)
        
        alt_baslik = QLabel("FOOTBALL EDITION • BASKETBALL EDITION • VOLLEYBALL EDITION")
        alt_baslik.setStyleSheet("font-size: 18px; color: #ffffff; background-color: rgba(0,0,0,100); padding: 5px; margin-bottom: 50px; font-weight: bold;")
        alt_baslik.setAlignment(Qt.AlignCenter)
        
        btn_basla = QPushButton("OYUNA BAŞLA")
        btn_basla.setCursor(Qt.PointingHandCursor)
        btn_basla.setFixedSize(280, 70)
        btn_basla.setStyleSheet("""
            QPushButton { background-color: #0d6efd; color: white; border: none; font-size: 24px; font-weight: bold; border-radius: 15px; }
            QPushButton:hover { background-color: #0256d9; }
        """)
        btn_basla.clicked.connect(lambda: self.ekranlar.setCurrentIndex(1)) 
        
        layout.addWidget(baslik)
        layout.addWidget(alt_baslik)
        layout.addWidget(btn_basla, alignment=Qt.AlignCenter)
        
        self.ekranlar.addWidget(widget) 

    def oyun_ekrani_kur(self):
        widget = QWidget()
        widget.setStyleSheet("background-color: transparent;") 
        ana_layout = QHBoxLayout(widget)
        
        self.scroll = QScrollArea()
        self.scroll.setWidgetResizable(True)
        self.scroll.setStyleSheet("border: none; background-color: transparent;")
        
        self.scroll_icerik = QWidget()
        self.scroll_icerik.setStyleSheet("background-color: transparent;")
        self.grid = QGridLayout(self.scroll_icerik)
        self.scroll.setWidget(self.scroll_icerik)
        
        sag_panel = QVBoxLayout()
        sag_panel.setAlignment(Qt.AlignTop)
        
        self.skor_lbl = QLabel("SKOR: 0 - 0")
        self.skor_lbl.setStyleSheet("font-size: 28px; font-weight: bold; color: #ffffff; background-color: rgba(0,0,0,100); padding: 5px; border-radius: 5px;")
        self.tur_lbl = QLabel("SIRADAKİ BRANŞ: FUTBOL")
        self.tur_lbl.setStyleSheet("font-size: 18px; color: #00ff00; font-weight: bold; margin-bottom: 20px;")
        
        self.detay_kutusu = QLabel("Oynamak için bir kart seçin.")
        self.detay_kutusu.setStyleSheet("background-color: white; border: 1px solid #dee2e6; border-radius: 10px; padding: 15px; color: #212529; font-size: 14px;")
        
        self.btn_oyna = QPushButton("SEÇİLİ KARTI OYNA")
        self.btn_oyna.setCursor(Qt.PointingHandCursor)
        self.btn_oyna.setStyleSheet("background-color: #0d6efd; color: white; border-radius: 8px; padding: 15px; font-size: 16px; font-weight: bold; margin-top: 20px;")
        self.btn_oyna.clicked.connect(self.oyna)
        
        sag_panel.addWidget(self.skor_lbl)
        sag_panel.addWidget(self.tur_lbl)
        sag_panel.addWidget(QLabel("<b><font color='white'>PUAN DETAYLARI:</font></b>"))
        sag_panel.addWidget(self.detay_kutusu)
        sag_panel.addWidget(self.btn_oyna)
        
        ana_layout.addWidget(self.scroll, 3) 
        ana_layout.addLayout(sag_panel, 1)   
        
        self.ekranlar.addWidget(widget) 

    def kartlari_ciz(self):
        for i in reversed(range(self.grid.count())): 
            self.grid.itemAt(i).widget().setParent(None)
        self.kart_widgetlari.clear()
        
        brans = self.sistem.tur_sirasi[self.sistem.aktif_tur] if self.sistem.aktif_tur < 12 else ""
        self.tur_lbl.setText(f"SIRADAKİ BRANŞ: {brans.upper()}")
        
        satir, sutun = 0, 0
        for i, k in enumerate(self.sistem.kullanici_kartlar):
            aktif_mi = (k.brans() == brans and k.enerji > 0)
            kart_ui = SporcuKarti(k, i, self, aktif_mi)
            self.grid.addWidget(kart_ui, satir, sutun)
            self.kart_widgetlari.append(kart_ui)
            
            sutun += 1
            if sutun > 2: 
                sutun = 0
                satir += 1

    def kart_secildi(self, indeks):
        self.secili_indeks = indeks
        for i, widget in enumerate(self.kart_widgetlari):
            widget.secili_yap(i == indeks)
            
        k = self.sistem.kullanici_kartlar[indeks]
        temel = k.ozellikler['P1']
        ceza = int(temel * 0.1) if k.enerji <= 70 else 0
        bonus = (k.seviye - 1) * 5
        toplam = k.performans_hesapla('P1', 70)
        
        self.detay_kutusu.setText(
            f"<b>{k.ad}</b><br>{k.takim}<br><br>"
            f"Temel Puan: <b>{temel}</b><br>"
            f"Enerji Cezası: <font color='red'>-{ceza}</font><br>"
            f"Seviye Bonusu: <font color='green'>+{bonus}</font><br><br>"
            f"<b style='font-size:18px;'>GÜNCEL GÜÇ: {toplam:.1f}</b>"
        )

    def oyna(self):
        if self.secili_indeks == -1:
            QMessageBox.warning(self, "Uyarı", "Lütfen önce bir kart seçin!")
            return
            
        k_kart = self.sistem.kullanici_kartlar[self.secili_indeks]
        brans = self.sistem.tur_sirasi[self.sistem.aktif_tur]
        
        if k_kart.brans() != brans:
            QMessageBox.warning(self, "Hata", f"Sadece {brans.upper()} kartı oynayabilirsiniz!")
            return
            
        pc_kart = self.ai.sec(self.sistem.pc_kartlar, brans)
        kp = k_kart.performans_hesapla("P1", self.sistem.moral)
        bp = pc_kart.performans_hesapla("P1", 70)
        
        if kp > bp:
            self.sistem.skorlar["Sen"] += 10
            msj, dr = "KAZANDIN!", "win"
        elif bp > kp:
            self.sistem.skorlar["PC"] += 10
            msj, dr = "KAYBETTİN!", "lose"
        else: msj, dr = "BERABERE", "draw"

        k_kart.enerji_guncelle(dr)
        k_kart.deneyim_guncelle(dr)
        
        QMessageBox.information(self, "Tur Bitti", f"{msj}\nSen: {kp:.1f} - PC: {bp:.1f}")
        
        self.sistem.aktif_tur += 1
        self.secili_indeks = -1 
        self.skor_lbl.setText(f"SKOR: {self.sistem.skorlar['Sen']} - {self.sistem.skorlar['PC']}")
        self.detay_kutusu.setText("Oynamak için bir kart seçin.")
        
        if self.sistem.aktif_tur >= 12:
            self.btn_oyna.setEnabled(False)
            winner = "SEN" if self.sistem.skorlar["Sen"] > self.sistem.skorlar["PC"] else "BİLGİSAYAR"
            QMessageBox.information(self, "Oyun Bitti", f"LİG BİTTİ!\nŞAMPİYON: {winner}")
        else:
            self.kartlari_ciz() 

if __name__ == "__main__":
    app = QApplication(sys.argv)
    pencere = OyunEkrani()
    pencere.show()
    sys.exit(app.exec_())

