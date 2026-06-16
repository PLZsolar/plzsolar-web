# plzsolar-web
import sys
import numpy as np
import matplotlib
matplotlib.use('QtAgg') # Konsolsuz çalışma için backend ayarı
from PyQt6.QtWidgets import (QApplication, QMainWindow, QWidget, QVBoxLayout, 
                             QHBoxLayout, QLabel, QComboBox, QLineEdit, 
                             QPushButton, QGroupBox, QFrame, QGridLayout, QSlider)
from PyQt6.QtGui import QFont, QIcon
from PyQt6.QtCore import Qt
from matplotlib.backends.backend_qt5agg import FigureCanvasQTAgg as FigureCanvas
from matplotlib.figure import Figure

# --- TÜRKİYE 81 İL TAM VERİ SETİ (kWh/kWp) ---
# Değerler yıllık ortalama günlük verimlerdir.
ILLER_VERI_SETI = {
    "Adana": 4.45, "Adıyaman": 4.30, "Afyonkarahisar": 3.90, "Ağrı": 4.10, "Aksaray": 4.10,
    "Amasya": 3.65, "Ankara": 3.85, "Antalya": 4.60, "Ardahan": 3.90, "Artvin": 2.80,
    "Aydın": 4.35, "Balıkesir": 3.75, "Bartın": 3.15, "Batman": 4.45, "Bayburt": 3.95,
    "Bilecik": 3.50, "Bingöl": 4.15, "Bitlis": 4.25, "Bolu": 3.30, "Burdur": 4.20,
    "Bursa": 3.55, "Çanakkale": 3.70, "Çankırı": 3.75, "Çorum": 3.70, "Denizli": 4.25,
    "Diyarbakır": 4.55, "Düzce": 3.25, "Edirne": 3.55, "Elazığ": 4.30, "Erzincan": 4.15,
    "Erzurum": 4.20, "Eskişehir": 3.90, "Gaziantep": 4.40, "Giresun": 2.90, "Gümüşhane": 3.75,
    "Hakkari": 4.40, "Hatay": 4.35, "Iğdır": 4.20, "Isparta": 4.20, "İstanbul": 3.40,
    "İzmir": 4.15, "Kahramanmaraş": 4.35, "Karabük": 3.45, "Karaman": 4.25, "Kars": 4.05,
    "Kastamonu": 3.35, "Kayseri": 4.10, "Kırıkkale": 3.85, "Kırklareli": 3.45, "Kırşehir": 3.95,
    "Kilis": 4.45, "Kocaeli": 3.35, "Konya": 4.15, "Kütahya": 3.85, "Malatya": 4.30,
    "Manisa": 4.15, "Mardin": 4.55, "Mersin": 4.60, "Muğla": 4.55, "Muş": 4.10,
    "Nevşehir": 4.05, "Niğde": 4.15, "Ordu": 2.95, "Osmaniye": 4.35, "Rize": 2.75,
    "Sakarya": 3.35, "Samsun": 3.10, "Siirt": 4.35, "Sinop": 3.05, "Sivas": 4.00,
    "Şanlıurfa": 4.70, "Şırnak": 4.45, "Tekirdağ": 3.45, "Tokat": 3.65, "Trabzon": 2.85,
    "Tunceli": 4.20, "Uşak": 4.00, "Van": 4.35, "Yalova": 3.40, "Yozgat": 3.85, "Zonguldak": 3.10
}

class MplCanvas(FigureCanvas):
    def __init__(self):
        fig = Figure(figsize=(6, 4), dpi=100, facecolor='#ffffff')
        self.axes = fig.add_subplot(111)
        fig.tight_layout()
        super().__init__(fig)

class PLZsolarApp(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("PLZsolar - GES Fizibilite ve Analiz")
        self.setMinimumSize(1100, 850)
        self.init_ui()

    def init_ui(self):
        central_widget = QWidget()
        self.setCentralWidget(central_widget)
        main_layout = QHBoxLayout(central_widget)

        # --- SOL PANEL (PARAMETRELER) ---
        sidebar = QVBoxLayout()
        
        # 1. Teknik Parametreler
        tech_group = QGroupBox("Teknik Parametreler")
        tech_lay = QGridLayout()
        
        self.il_combo = QComboBox()
        self.il_combo.addItems(sorted(ILLER_VERI_SETI.keys()))
        
        self.kwp_input = QLineEdit("10")
        
        self.loss_slider = QSlider(Qt.Orientation.Horizontal)
        self.loss_slider.setRange(5, 30)
        self.loss_slider.setValue(15)
        self.loss_label = QLabel("%15")
        self.loss_slider.valueChanged.connect(lambda v: self.loss_label.setText(f"%{v}"))
        
        tech_lay.addWidget(QLabel("Şehir:"), 0, 0)
        tech_lay.addWidget(self.il_combo, 0, 1)
        tech_lay.addWidget(QLabel("Sistem Gücü (kWp):"), 1, 0)
        tech_lay.addWidget(self.kwp_input, 1, 1)
        tech_lay.addWidget(QLabel("Sistem Kayıpları:"), 2, 0)
        tech_lay.addWidget(self.loss_slider, 2, 1)
        tech_lay.addWidget(self.loss_label, 2, 2)
        tech_group.setLayout(tech_lay)

        # 2. Finansal Parametreler
        fin_group = QGroupBox("Finansal Parametreler")
        fin_lay = QGridLayout()
        
        self.cost_per_kwp = QLineEdit("750")
        self.elec_price = QLineEdit("4.50") 
        self.usd_try = QLineEdit("32.50")
        
        fin_lay.addWidget(QLabel("Kurulum Maliyeti ($/kWp):"), 0, 0)
        fin_lay.addWidget(self.cost_per_kwp, 0, 1)
        fin_lay.addWidget(QLabel("Elek. Fiyatı (TL/kWh):"), 1, 0)
        fin_lay.addWidget(self.elec_price, 1, 1)
        fin_lay.addWidget(QLabel("Dolar Kuru (TL):"), 2, 0)
        fin_lay.addWidget(self.usd_try, 2, 1)
        fin_group.setLayout(fin_lay)

        self.btn = QPushButton("HESAPLA VE ANALİZ ET")
        self.btn.setFixedHeight(50)
        self.btn.setStyleSheet("background-color: #f39c12; color: white; font-weight: bold; font-size: 14px; border-radius: 5px;")
        self.btn.clicked.connect(self.analiz_et)

        sidebar.addWidget(tech_group)
        sidebar.addWidget(fin_group)
        sidebar.addStretch()
        sidebar.addWidget(self.btn)

        # --- SAĞ PANEL ---
        display_panel = QVBoxLayout()
        self.canvas = MplCanvas()
        display_panel.addWidget(self.canvas)
        
        self.res_card = QFrame()
        self.res_card.setStyleSheet("background-color: #fdfdfd; border: 1px solid #ddd; border-radius: 10px;")
        self.res_lay = QVBoxLayout(self.res_card)
        self.res_text = QLabel("Analiz yapmak için parametreleri girip butona basın.")
        self.res_text.setAlignment(Qt.AlignmentFlag.AlignTop)
        self.res_text.setWordWrap(True)
        self.res_lay.addWidget(self.res_text)
        
        display_panel.addWidget(self.res_card)
        main_layout.addLayout(sidebar, 1)
        main_layout.addLayout(display_panel, 2)

    def analiz_et(self):
        try:
            il = self.il_combo.currentText()
            kwp = float(self.kwp_input.text())
            kayip = self.loss_slider.value() / 100
            maliyet_birim = float(self.cost_per_kwp.text())
            elek_tl = float(self.elec_price.text())
            kur = float(self.usd_try.text())
            
            # Hesaplama
            verim = ILLER_VERI_SETI[il]
            yillik_net_uretim = kwp * verim * 365 * (1 - kayip)
            toplam_maliyet_tl = kwp * maliyet_birim * kur
            yillik_kazanc_tl = yillik_net_uretim * elek_tl
            amortisman = toplam_maliyet_tl / yillik_kazanc_tl if yillik_kazanc_tl > 0 else 0

            # Grafik
            yillar = np.arange(0, 11)
            nakit_akisi = (yillar * yillik_kazanc_tl) - toplam_maliyet_tl
            self.canvas.axes.cla()
            self.canvas.axes.axhline(0, color='black', lw=1)
            self.canvas.axes.plot(yillar, nakit_akisi, marker='o', color='#e67e22', label="Kümülatif Net Kar")
            self.canvas.axes.fill_between(yillar, nakit_akisi, 0, where=(nakit_akisi >= 0), color='green', alpha=0.2)
            self.canvas.axes.set_title(f"PLZsolar: {il} Yatırım Geri Dönüşü")
            self.canvas.axes.set_xlabel("Yıl")
            self.canvas.axes.set_ylabel("Net Bakiye (TL)")
            self.canvas.axes.grid(True, linestyle=':', alpha=0.6)
            self.canvas.draw()

            # Rapor
            self.res_text.setText(
                f"<h2 style='color:#e67e22;'>PLZsolar Fizibilite Özeti</h2>"
                f"<b>Seçilen Şehir:</b> {il}<br>"
                f"<b>Toplam Yatırım Bedeli:</b> {toplam_maliyet_tl:,.2f} TL<br>"
                f"<b>Yıllık Net Üretim:</b> {yillik_net_uretim:,.0f} kWh<br>"
                f"<b>Yıllık Tasarruf:</b> {yillik_kazanc_tl:,.2f} TL<br><br>"
                f"<span style='font-size:16px;'><b>TAHMİNİ AMORTİSMAN: {amortisman:.1f} YIL</b></span>"
            )

        except Exception as e:
            self.res_text.setText(f"Hata: {str(e)}")

if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = PLZsolarApp()
    window.show()
    sys.exit(app.exec())