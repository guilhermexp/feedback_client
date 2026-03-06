# Client Feedback Mode

Sistema completo de feedback visual para clientes em aplicacoes React/Next.js.

Permite que clientes cliquem em elementos da pagina, deixem comentarios, e enviem o feedback como markdown estruturado — direto para o Suna como nota vinculada ao projeto.

## Como funciona

1. Usuario ativa o toggle "Modo Feedback" nas configuracoes do app
2. Botao "Feedback" aparece no canto inferior direito
3. Clica para ativar modo de anotacao
4. Clica em elementos da pagina e deixa comentarios
5. Clica "Enviar" → feedback vai pro Suna como nota vinculada ao projeto

## Integracao Suna

O feedback eh enviado automaticamente para o Suna como nota vinculada ao projeto via endpoint `POST /notes/capture`.

### Variaveis de ambiente

```env
SUNA_API_URL=https://suna-api.claudedokploy.com/api
SUNA_API_KEY=PUBLIC_KEY:SECRET_KEY
SUNA_PROJECT_ID=uuid-do-projeto-no-suna
```

- `SUNA_API_KEY`: Criar em Suna > Settings > API Keys (formato `public:secret`)
- `SUNA_PROJECT_ID`: ID do projeto no Suna (UUID)

## Instalacao

Consulte o arquivo `SKILL.md` para instrucoes completas de integracao.

## Stack

- React 18+
- Next.js 13+ (opcional)
- Zustand (recomendado) ou localStorage
- Biblioteca `agentation` para anotacoes visuais
- Suna Notes API para armazenamento vinculado a projetos
