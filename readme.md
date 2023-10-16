
# Darbo su nestruktūrizuotais failais metodika
by Linas Kapočius

https://www.google.com/url?sa=i&url=https%3A%2F%2Fwww.whitebearlake.org%2Fee%2Fpage%2Ftrash-treasure-day&psig=AOvVaw2iv3GhcOcUBhGBCaYfW73U&ust=1695461436374000&source=images&cd=vfe&opi=89978449&ved=0CBAQjRxqFwoTCMje45f0vYEDFQAAAAAdAAAAABAE


![image](https://github.com/DataAIchemist/palantir_nestrukt/assets/68922285/6e00418a-e1fa-43c8-8a80-48393a116f00)

Pirma vieta, kur reikia ieškoti informacijos kaip tvarkyti nestruktūrizuotus failus yra:
https://www.palantir.com/docs/foundry/transforms-python/unstructured-files/index.html 
FHIR: https://www.fhir.org/


## Failo struktūros dokumentacija/paieška:
* Oficiali dokumentacija. Pvz., https://parquet.apache.org/docs/ 
* https://www.registrai.lt - informacija apie valstybines IS
*  https://www.fileformat.com/ - informacija apie failų formatus ir jų dokumentacija
* Žiniatinkliai (google.lt, etc.), LLM AI modeliai (ChatGPT, Bing AI, etc.), forumai (stackoverflow, dev.to, etc.)
* Reverse Engineering - išimtinais atvejais, pravartu paklausti kolegų, ar su tokiais failų formatais neteko dirbti.


## Failių peržiūra arba truputį reverse enginnering

Dirbant su big data, daug failų. Pirmiausia reikėtų paimtį mažą reprezentacinę dalį duomenų (keletą failų) ir iš jų sudaryti naują Palantir failų sistemos duomenų rinkinį (datafram'ą).

Pavyzdžiui, turime apsaugotą slaptažodžiu archyvą, kuriame yra
labai daug nestruktūrizuotų .xml failiukų. Prieš pradedant darbą 
svarbu pasiekti duomenis per Palantir failų sistemą ir nusifiltruoti pagal atitinkamus kriterijus. Kaip šiuo atveju atrinkome tik tuos failus, kurių pavadinimas prasideda "vda". 
```
from transforms.api import transform, Input, Output
from transforms.external.systems import use_external_systems, Credential, EgressPolicy, ExportControl
import py7zr
import shutil
import tempfile


@use_external_systems(
    export_control=ExportControl(markings=[]),
    egress=EgressPolicy('ri.resource-policy-manager.global.network-egress-policy'),
    creds=Credential("ri.credential..credential")
)
@transform(
    output=Output("ri.foundry.main.dataset"),
    source_df=Input("ri.foundry.main.dataset"),
)
def compute(export_control, egress, creds, source_df, output):
    out_fs = output.filesystem()
    in_fs = source_df.filesystem()
    password = creds.get("password")

    def process_file(file_status):
        with in_fs.open(file_status.path, 'rb') as raw_file:
            with tempfile.NamedTemporaryFile() as tmp:
                shutil.copyfileobj(raw_file, tmp)
                tmp.flush()
                with py7zr.SevenZipFile(tmp.name, mode='r', password=password) as zip_file:
                    with tempfile.TemporaryDirectory() as tmp_dir:
                        zip_file.extractall(path=tmp_dir)
                        full_file_names = zip_file.getnames()

                        for file_name in full_file_name:
			                if file_name.startswith("vda"):
                                with open(f'{tmp_dir}/{file_name}', 'rb') as tmp_file:
                                    with out_fs.open(file_name, 'wb') as f:
                                        shutil.copyfileobj(tmp_file, f)
                                        f.flush()


    in_fs.files().foreach(process_file)
```

#### Jeigu vis dar failo formatas nėra aiškus/suprantamas.

Tarkime archyve yra dar vienas archyvas su nežinomais failiukais. Uždedamas breakpoint'as ir perskaitoma jo struktūra repozitorijos terminale
```
 def process_file(file_status):
        name = file_status.path.split(".")[0]
        with tempfile.TemporaryDirectory() as tmp:
            with in_fs.open(file_status.path, 'rb') as raw_file:
                with open(f"{tmp}/{name}", "wb") as f:
                    shutil.copyfileobj(raw_file, f)
                    f.flush()

            with zipfile.ZipFile(f"{tmp}/{name}", 'r') as zip_file:
                with tempfile.TemporaryDirectory() as tmp_dir:
                    zip_file.extractall(path=tmp_dir)
                    for filename in zip_file.namelist():
                        if not filename.endswith(".txt.crc"):  # skip .txt.crc files
                            zip_item = zip_file.getinfo(filename)
                            if zip_item.is_dir():
                                continue  # skip directories
                            file_content = zip_file.read(filename) # <--- breakpoint
```

Struktūra analizuojama lokaliai pasirinktais būdais.


