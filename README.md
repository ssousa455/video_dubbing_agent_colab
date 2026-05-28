# Video Dubbing Agent — Colab

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/ssousa455/video_dubbing_agent_colab/blob/main/video_dubbing_agent_colab.ipynb)

Tradução e dublagem automática de vídeos usando IA. Basta enviar um vídeo e escolher o idioma de destino — o sistema transcreve, traduz e gera áudio dublado sincronizado.

Baseado no Space [sauravghu/video-dubbing-agent](https://huggingface.co/spaces/sauravghu/video-dubbing-agent) do Hugging Face, adaptado para rodar no Google Colab.

## Como usar (passo a passo)

1. Clique no botão **"Open in Colab"** acima
2. No Colab, vá em **Runtime > Run all** (ou pressione `Ctrl+F9`)
3. Aguarde as instalações (pode levar alguns minutos na primeira vez)
4. Na seção de upload, selecione o vídeo do seu computador
5. Escolha o idioma de destino e clique em **Run**
6. O vídeo dublado será baixado automaticamente

## Limitações

- **Tamanho máximo do arquivo: 100 MB** (limite do Google Colab para upload)
- O tempo de processamento depende do tamanho do vídeo e da GPU alocada
- Funciona melhor com vídeos curtos (até ~10 minutos)
- A qualidade da dublagem depende da qualidade do áudio original

## Requisitos

- Conta gratuita no [Google Colab](https://colab.research.google.com/)
- Navegador moderno (Chrome, Firefox, Edge)
- Não é necessário instalar nada no seu computador

## Licença

Este projeto é uma adaptação do [video-dubbing-agent](https://huggingface.co/spaces/sauravghu/video-dubbing-agent) de Saurav Ghosh, licenciado sob as licenças dos componentes originais.
