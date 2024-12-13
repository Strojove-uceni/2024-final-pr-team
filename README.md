# TabuVision

## Cíl projektu
Hlavním cílem projektu je vytvořit software umožňující s co nejvyšší úspěšností (a pokud možno v přijetelném výpočetním čase) extrahovat tabulky, obsahující relevatní finanční informace, z výkazů zisků a ztrát volně dostupných na webu Ministerstva spravedlnosti https://justice.cz/

## Postup
Celý postup se dá shrnout do 5 kroků:
1) Korekce orientace stránky
2) Detekce tabulky na stránce
3) Korekce skew tabulky (napřímení do horizontální polohy)
4) Detekce struktury tabulky (sloupce a řádky)
5) Rozpoznání textu uvnitř jednotlivých buněk tabulky

### Korekce orientace stránky
Pro korekci orientace stránky jsme nejprve vyzkoušeli Pytesseract (python wrapper pro Google Tesseract-OCR engine), konkrétně funkci `image_to_osd`. Pytesseract nakonec dosáhl tak excelentních výsledků, že nebylo potřeba ladit jiné (zpravidla co do úspěšnosti horší) modely na rozpoznávání orientace skriptu. Klíčové bylo správně obrázek předzpracovat. Vlivem správného předzpracování obrázku jsme docílíli zlepšení úspěšnosti o 13,45 procentních bodů (z 85,68% na 99,13%). Využívali jsme Pytesseract `image_to_osd` s Page Segmentation Mode 0 (PSM 0). Nejprve jsme zkoušeli vkládat nepředzpracované obrázky již vyříznutých tabulek. To se neosvědčilo. Poté jsme zkoušeli vkládat celé stránky různě předzpracované. Výsledky ukazuje následující tabulka.

| **PyTesseract (PSM 0)**                        | **accuracy [%]** |
|------------------------------------------------|------------------:|
| celé stránky (Sauvola - w = 15, k = 0.2, lang = ces)       | 99.13            |
| celé stránky (Sauvola - w = 15, k = 0.2)       | 99.10            |
| celé stránky (Sauvola - w = 15, k = 0.3)       | 98.97            |
| celé stránky (Sauvola - w = 15, k = 0.05)      | 98.90            |
| celé stránky (Sauvola - w = max(img_width,img_height), k = 0.2) | 98.15            |
| celé stránky (Sauvola) + korekce skew          | 96.97            |
| celé stránky (RGB) + korekce skew              | 94.12            |
| celé stránky (RGB)                             | 93.77            |
| celé stránky (Gaussian Blur 5x5 + Sauvola)     | 87.47            |
| výřezy tabulek (RGB)                           | 85.68            |

Kde Sauvola je metoda adaptivního prahování (w = window size, k = parametr metody), korekce skew znamená, že jsme stránku nejprve narovnali pomocí algoritmu projekčního profilování.

Celý testovací dataset je k dispozici zde: <a href="https://drive.google.com/drive/folders/1p_eGllQDmgyUTWAXq2Z7LjEKfZ6MTLlS?usp=drive_link" target="_blank">Rotation dataset</a>
Dataset obsahuje 6757 obrázků. Obrázky byly vytažené z PDF souborů výkazů zisků a ztrát stažených z webu Ministerstva spravedlnosti. Jednotlivé obrázky jsme manuálně prošli a zrotovali (o 0,90,180, nebo 270 stupňů) do standardní polohy. Poté jsme přidali náhodně kvadraturní rotaci (0,90,180, nebo 270 stupňů). Hodnotu rotace jsme uložili do jména obrázku. Testování probíhalo tak, že jsme přičetli +1 za každou správně uhodnutou rotaci pomocí Pytesseract `image_to_osd`. Accuracy pak vznikla jako počet správně predikovaných rotací / počet obrázků. 
### Detekce tabulky

#### Datasety

##### PubTables1M - Detection
Jakožto dataset pro účely detekce tabulky v obrázku dokumentu jsme využili rozsáhlý dataset PubTables1M-Detection (https://huggingface.co/datasets/bsmock/pubtables-1m/tree/main) čítající 575305 obrázků stránek dokumentů, ve kterých jsou anotovány tabulky ve formě bounding boxů do dvou tříd "table" a "rotated_table". Obrázky jsou sesbírané především z věděckých článků. Obrázky a anotace z tohoto datasetu jsou rozděleny na train/val/test v poměru 460589/57591/57125 vzorků. Tento dataset jsem využili k předtrénování některých modelů. Takto předtrénované modely jsou později označeny jako "fine-tuned".

##### Náš dataset
Jako druhý dataset jsme manuálně anotovali tabulky v 6676 obrázcích, které jsme získali ze stažených výkazů zisků a ztrát ze stránek Ministerstva spravedlnosti https://justice.cz/. Tyto výkazy jsou PDF dokumenty obsahující naskenované stránky výkazů nebo stránky v digitální podobě. Tabulky jsme anotovali v prostředí CVAT pomocí bounding boxů a jednou třídou "table". Zhruba 10-20% obrázků v datasetu neobsahuje žádnou tabulku.

#### Metody

##### Klasické metody zpracování obrazu
Prvně jsme se pokusili detekovat tabulky pomocí klasických metod zpracování obrazu. K segmentaci textu a mřížky tabulky jsme využili metodu Sauvola adaptive thresholding. Dále jsme pak potřebovali extrahovat mřížku tabulky. Předpokládali jsme, že tabulka má mřížku a samotná tabulky je dokonale narovnána do standardní polohy. Poté jsme použili morfologický opening s vektory jedniček 1x40 a také 40x1. Tyto dva výstupy z openingů jsme pak zkombinovali do jednoho výstupu pomocí bitového OR (sjednocení bílých pixelů). Následně jsme pak pomocí funkce findContours z OpenCV našli externí kontury a těm určily minimální ohraničující obdélník. Tento obdélník jsme pak považovali za bounding box tabulky s třídou "table". Výrazným omezením této metody jsou předpoklady dokonale standardní orientace tabulky a také, že tabulky obsahuje mřížku. Takové tabulky se ovšem v datasetu vyskytují zřídka (odhadem 30% tabulek).

##### Metody založené na hlubokém učení
Vyzkoušeli jsme celkem 4 architektury modelů standardně používané pro Object detection úkoly. Jsou to YOLOv8 Medium, YOLOv11 Large, RT-DETR Large a Faster-RCNN.

### Korekce rotace a skew

#### Korekce rotace

Pro korekci rotace jsme 

### Detekce struktury

### Rozpoznání textu

## Shrnutí
## Odkazy
Většina použitých kódů je dostupná ke stažení zde: <a href="https://drive.google.com/drive/folders/1VXDQpVIAQ3XSNKVEbv4OpxZIaYCLkBbK?usp=sharing" target="_blank">TabuVision Codes</a>

