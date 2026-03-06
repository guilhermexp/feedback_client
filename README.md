# Client Feedback Mode

Sistema completo de feedback visual para clientes em aplicacoes React/Next.js.

Permite que clientes cliquem em elementos da pagina, deixem comentarios, e enviem o feedback como markdown estruturado.

## Como funciona

1. Usuario ativa o toggle "Modo Feedback" nas configuracoes do app
2. Botao "Feedback" aparece no canto inferior direito
3. Clica para ativar modo de anotacao
4. Clica em elementos da pagina e deixa comentarios
5. Exporta como `.md`, copia, ou envia via API

## Instalacao

Consulte o arquivo `SKILL.md` para instrucoes completas de integracao.

## Stack

- React 18+
- Next.js 13+ (opcional)
- Zustand (recomendado) ou localStorage
- Biblioteca `agentation` para anotacoes visuais
