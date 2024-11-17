# TabuVision

## Cíle projektu
1) Hlavním cílem projektu je extrahovat tabulky, obsahující relevatní informace, z výkazů zisků a ztrát volně dostupných na webu Ministerstva spravedlnosti https://justice.cz/

## Postup
Celý postup se dá shrnout do 4 kroků:
1) Detekce tabulky
2) Korekce rotace abulky (napřímení do standardní polohy)
3) Detekce struktury tabulky (sloupce a řádky)
4) Rozpoznání textu uvnitř jednotlivých buněk tabulky

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
