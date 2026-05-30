Per far funzionare un progetto dove si usa un export c++ rnbo bisogna:

1 - Copiare la cartella "rnbo.juce.example-main" dentro la repo (oppure installare col comando seguente, così hai già anche i moduli): 

`
git clone --branch juce8 --recurse-submodules https://github.com/Cycling74/rnbo.example.juce.git
`

2 - Esportare il patcher rnbo dentro la cartella `rnbo.juce.example-main/export` (C'è anche un `readme` che lo dice dentro) (eportarlo in c++ con coi nomi di default)

3 - Nel `rnbo.juce.example-main/README.md` ci sono i prerequisiti di codice da installare.

4 - inizializzare il submodulo juce in qualche modo (non c'è bisogno se hai clonato nel passaggio 1)

5 - Per creare il codice "tramite" tra la UI (Web) e il DSP (RNBO) utilizza come contesto i file:

`rnbo_juce_webui_prompt_instructions.md`
`rnbo.juce.example-main/CUSTOM_UI.md`
`rnbo.juce.example-main/README.md`

prompt iniziale:

Ciao, devi costruire un VST3 a partire da un export c++ in rnbo. L'export rnbo lo trovi già nella cartella "rnbo.example.juce/export". Leggi bene tutte le istruzioni dei file .md nel progetto. In particolare @file:rnbo_juce_webui_prompt_instructions.md e #file:CUSTOM_UI.md .

Stiamo costruendo un plugin {spiega che tipo di plugin è, dai contesto sui parametri e su quello che deve fare}

param nome valoredef @min N @max N
param nome valoredef @min N @max N

Il vst3 si chiamerà "NomePlugin_N"

UI development:

se il progetto viene costruito secondo il prompt è anche possibile lanciare nella cartella dove c'è il pacchetto npm il comando "run dev" per far girare la cosa su localhost, a quel punto il plugin quando viene lanciato non va in fallback sulla versione /dist dell'interfaccia (che si costruisce con il comando "run build") ma va sulla versione di development. In questa maniera LIVE mentre il plugin gira è possibile modificarne l'interfaccia.

Production commands:

Eseguire dalla cartella rnbo.example.juce/

1. Bumpa la versione in Plugin.cmake (PRODUCT_NAME e PLUGIN_CODE) prima di ogni build

2. Pulisci la build precedente e riconfigura
rm -rf build
cmake -S . -B build -G Ninja -DRNBO_EDITOR_MODE=WEBVIEW

1. Builda la web UI (obbligatorio prima del plugin)
cd src/webui && npm run build && cd ../..

1. Compila il plugin
cmake --build build

1. Copia il bundle nella cartella VST3 (sostituisci NomePlugin_N con il nome effettivo)
ditto build/RNBOAudioPlugin_artefacts/Debug/VST3/NomePlugin_N.vst3 \
      /Users/pierpaoloovarini/Desktop/VST3/NomePlugin_N.vst3

1. Firma il bundle (ad-hoc, sufficiente per uso locale)
codesign --force --deep --sign - \
         /Users/pierpaoloovarini/Desktop/VST3/NomePlugin_N.vst3

1. Verifica la firma
codesign --verify --deep --strict --verbose=4 \
         /Users/pierpaoloovarini/Desktop/VST3/NomePlugin_N.vst3

Nota: se vuoi aggiornare solo la UI senza cambiare C++:
- Modifica i file in src/webui/
- Esegui solo i passi 3 e 4 (npm run build + cmake --build build)
- Poi i passi 5, 6, 7 per ridistribuire
- Ricorda comunque di fare un nuovo PRODUCT_NAME/PLUGIN_CODE per ogni bundle che distribuisci