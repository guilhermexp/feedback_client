---
name: client-feedback-mode
description: Use when adding visual client feedback annotation system to React or Next.js applications. Triggers on requests to add feedback collection, client review mode, annotation system, or visual commenting to a web app. Always integrates a toggle in the project's settings page with initial state disabled.
---

# Client Feedback Mode

## Overview

Sistema completo de feedback visual para clientes. Permite que clientes cliquem em elementos da pagina, deixem comentarios, e enviem o feedback como markdown estruturado.

Usa a biblioteca `agentation` como base para anotacoes visuais e adiciona painel lateral, exportacao markdown, e envio para API.

**Estado inicial: DESATIVADO.** O feedback so aparece quando o usuario ativa via toggle nas configuracoes do app.

## When to Use

- Adicionar sistema de feedback/revisao visual em app React ou Next.js
- Cliente precisa anotar elementos da UI com comentarios
- Precisa de exportacao de feedback em markdown
- Quer botao de feedback no canto inferior direito da aplicacao

## Quick Reference

| Prop | Tipo | Default | Descricao |
|------|------|---------|-----------|
| title | string | "Client Feedback" | Titulo no markdown exportado |
| includeMetadata | boolean | true | Incluir hora/posicao no export |

## Installation Steps

### 1. Instalar dependencia

Detectar o package manager do projeto (bun/pnpm/yarn/npm) e rodar:

```bash
bun add agentation
```

### 2. Criar store de estado persistente

Detectar o gerenciador de estado do projeto. Se usar Zustand, criar um store com `persist` middleware. Se nao usar Zustand, criar um wrapper com localStorage.

**Zustand (recomendado):**

```typescript
// src/stores/feedbackStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface FeedbackState {
  enabled: boolean;
  setEnabled: (enabled: boolean) => void;
  toggle: () => void;
}

export const useFeedbackStore = create<FeedbackState>()(
  persist(
    (set) => ({
      enabled: false,
      setEnabled: (enabled) => set({ enabled }),
      toggle: () => set((state) => ({ enabled: !state.enabled })),
    }),
    { name: 'feedback-mode' }
  )
);
```

**Sem Zustand (localStorage puro):**

```typescript
// src/feedback/use-feedback-store.ts
import { useState, useEffect, useCallback } from 'react';

const STORAGE_KEY = 'feedback-mode';

export function useFeedbackStore() {
  const [enabled, setEnabledState] = useState(() => {
    try { return localStorage.getItem(STORAGE_KEY) === 'true'; }
    catch { return false; }
  });

  const setEnabled = useCallback((value: boolean) => {
    setEnabledState(value);
    localStorage.setItem(STORAGE_KEY, String(value));
  }, []);

  const toggle = useCallback(() => {
    setEnabled(!enabled);
  }, [enabled, setEnabled]);

  return { enabled, setEnabled, toggle };
}
```

Se o projeto tiver um barrel export de stores (ex: `src/stores/index.ts`), adicionar o export do novo store la.

### 3. Criar arquivos em src/feedback/

Criar a pasta `src/feedback/` com os 4 arquivos abaixo.

#### src/feedback/generate-markdown.ts

```typescript
interface AnnotationData {
  id: string;
  comment: string;
  element: string;
  elementPath: string;
  x: number;
  y: number;
  timestamp: number;
  selectedText?: string;
  intent?: 'fix' | 'change' | 'question' | 'approve';
  severity?: 'blocking' | 'important' | 'suggestion';
}

interface GenerateMarkdownOptions {
  title?: string;
  pageUrl?: string;
  includeMetadata?: boolean;
}

export function generateMarkdown(
  annotations: AnnotationData[],
  options: GenerateMarkdownOptions = {}
): string {
  const {
    title = 'Client Feedback',
    pageUrl = typeof window !== 'undefined' ? window.location.href : '',
    includeMetadata = true,
  } = options;

  const now = new Date();
  const dateStr = now.toISOString().split('T')[0];
  const timeStr = now.toTimeString().split(' ')[0];

  let md = `# ${title}\n\n`;
  md += `- **Pagina:** ${pageUrl}\n`;
  md += `- **Data:** ${dateStr} ${timeStr}\n`;
  md += `- **Total de anotacoes:** ${annotations.length}\n`;

  const blocking = annotations.filter((a) => a.severity === 'blocking').length;
  const important = annotations.filter((a) => a.severity === 'important').length;
  const suggestion = annotations.filter((a) => a.severity === 'suggestion').length;

  if (blocking || important || suggestion) {
    md += `- **Blocking:** ${blocking} | **Important:** ${important} | **Suggestion:** ${suggestion}\n`;
  }

  md += `\n---\n\n`;

  annotations.forEach((annotation, index) => {
    const num = index + 1;
    const intentLabel = annotation.intent ? ` [${annotation.intent.toUpperCase()}]` : '';
    const severityLabel = annotation.severity ? ` (${annotation.severity})` : '';

    md += `## ${num}. ${annotation.comment}${intentLabel}${severityLabel}\n\n`;
    md += `- **Elemento:** \`${annotation.element}\`\n`;
    md += `- **Caminho:** \`${annotation.elementPath}\`\n`;

    if (annotation.selectedText) {
      md += `- **Texto selecionado:** "${annotation.selectedText}"\n`;
    }

    if (includeMetadata) {
      const ts = new Date(annotation.timestamp).toLocaleTimeString();
      md += `- **Hora:** ${ts}\n`;
      md += `- **Posicao:** (${Math.round(annotation.x)}, ${Math.round(annotation.y)})\n`;
    }

    md += `\n`;
  });

  return md;
}

export function downloadMarkdown(
  annotations: AnnotationData[],
  options: GenerateMarkdownOptions = {}
): void {
  const md = generateMarkdown(annotations, options);
  const blob = new Blob([md], { type: 'text/markdown;charset=utf-8' });
  const url = URL.createObjectURL(blob);

  const dateStr = new Date().toISOString().split('T')[0];
  const filename = `feedback-${dateStr}.md`;

  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}

export function copyMarkdownToClipboard(
  annotations: AnnotationData[],
  options: GenerateMarkdownOptions = {}
): Promise<void> {
  const md = generateMarkdown(annotations, options);
  return navigator.clipboard.writeText(md);
}
```

#### src/feedback/feedback-panel.tsx

```tsx
import React from 'react';

interface Annotation {
  id: string;
  comment: string;
  element: string;
  elementPath: string;
  x: number;
  y: number;
  timestamp: number;
  selectedText?: string;
  intent?: string;
  severity?: string;
}

interface FeedbackPanelProps {
  annotations: Annotation[];
  onExport: () => void;
  onCopy: () => void;
  onClear: () => void;
  onClose: () => void;
  onDeleteAnnotation: (id: string) => void;
  onSend: () => Promise<void>;
  isSending?: boolean;
  sendStatus?: 'idle' | 'success' | 'error';
}

export function FeedbackPanel({
  annotations,
  onExport,
  onCopy,
  onClear,
  onClose,
  onDeleteAnnotation,
  onSend,
  isSending = false,
  sendStatus = 'idle',
}: FeedbackPanelProps) {
  return (
    <div style={styles.overlay}>
      <div style={styles.panel}>
        <div style={styles.header}>
          <span style={styles.title}>Feedback ({annotations.length})</span>
          <button onClick={onClose} style={styles.closeBtn}>X</button>
        </div>

        <div style={styles.list}>
          {annotations.length === 0 ? (
            <p style={styles.empty}>
              Nenhuma anotacao ainda. Clique em elementos da pagina para adicionar feedback.
            </p>
          ) : (
            annotations.map((ann, i) => (
              <div key={ann.id} style={styles.card}>
                <div style={styles.cardHeader}>
                  <span style={styles.cardNum}>#{i + 1}</span>
                  <code style={styles.cardElement}>{ann.element}</code>
                  <button
                    onClick={() => onDeleteAnnotation(ann.id)}
                    style={styles.deleteBtn}
                    title="Remover anotacao"
                  >
                    X
                  </button>
                </div>
                <p style={styles.cardComment}>{ann.comment}</p>
                {ann.selectedText && (
                  <p style={styles.cardSelected}>&ldquo;{ann.selectedText}&rdquo;</p>
                )}
                <div style={styles.cardMeta}>
                  <code style={styles.cardPath}>{ann.elementPath}</code>
                </div>
              </div>
            ))
          )}
        </div>

        {annotations.length > 0 && (
          <div style={styles.actions}>
            <button
              onClick={onSend}
              style={{
                ...styles.sendBtn,
                opacity: isSending ? 0.7 : 1,
                cursor: isSending ? 'not-allowed' : 'pointer',
              }}
              disabled={isSending}
            >
              {isSending ? 'Enviando...' : sendStatus === 'success' ? 'Enviado!' : 'Enviar'}
            </button>
            <button onClick={onExport} style={styles.exportBtn}>.md</button>
            <button onClick={onCopy} style={styles.copyBtn}>Copiar</button>
            <button onClick={onClear} style={styles.clearBtn}>Limpar</button>
          </div>
        )}
      </div>
    </div>
  );
}

const styles: Record<string, React.CSSProperties> = {
  overlay: { position: 'fixed', top: 0, right: 0, bottom: 0, width: '380px', zIndex: 99998, pointerEvents: 'auto' },
  panel: { height: '100%', background: '#1a1a2e', color: '#e0e0e0', display: 'flex', flexDirection: 'column', boxShadow: '-4px 0 20px rgba(0,0,0,0.3)', fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif', fontSize: '14px' },
  header: { display: 'flex', justifyContent: 'space-between', alignItems: 'center', padding: '16px 20px', borderBottom: '1px solid #2a2a4a', flexShrink: 0 },
  title: { fontWeight: 700, fontSize: '16px' },
  closeBtn: { background: 'none', border: 'none', color: '#888', fontSize: '18px', cursor: 'pointer', padding: '4px 8px', borderRadius: '4px' },
  list: { flex: 1, overflowY: 'auto', padding: '12px 16px' },
  empty: { color: '#666', textAlign: 'center', padding: '40px 20px', lineHeight: 1.6 },
  card: { background: '#16213e', borderRadius: '8px', padding: '12px 16px', marginBottom: '10px', border: '1px solid #2a2a4a' },
  cardHeader: { display: 'flex', alignItems: 'center', gap: '8px', marginBottom: '8px' },
  cardNum: { color: '#7c83ff', fontWeight: 700, fontSize: '13px', flexShrink: 0 },
  cardElement: { background: '#0f3460', padding: '2px 8px', borderRadius: '4px', fontSize: '12px', color: '#a0c4ff', flex: 1, overflow: 'hidden', textOverflow: 'ellipsis', whiteSpace: 'nowrap' },
  deleteBtn: { background: 'none', border: 'none', color: '#666', cursor: 'pointer', fontSize: '14px', padding: '2px 6px', borderRadius: '4px', flexShrink: 0 },
  cardComment: { margin: '0 0 6px', lineHeight: 1.5 },
  cardSelected: { margin: '0 0 6px', color: '#888', fontStyle: 'italic', fontSize: '13px' },
  cardMeta: { marginTop: '4px' },
  cardPath: { fontSize: '11px', color: '#555', wordBreak: 'break-all' },
  actions: { display: 'flex', gap: '8px', padding: '16px 20px', borderTop: '1px solid #2a2a4a', flexShrink: 0 },
  sendBtn: { flex: 2, padding: '10px', background: '#2ecc71', color: '#fff', border: 'none', borderRadius: '6px', fontWeight: 600, cursor: 'pointer', fontSize: '13px' },
  exportBtn: { flex: 1, padding: '10px', background: '#7c83ff', color: '#fff', border: 'none', borderRadius: '6px', fontWeight: 600, cursor: 'pointer', fontSize: '13px' },
  copyBtn: { padding: '10px 16px', background: '#2a2a4a', color: '#e0e0e0', border: 'none', borderRadius: '6px', fontWeight: 600, cursor: 'pointer', fontSize: '13px' },
  clearBtn: { padding: '10px 16px', background: 'transparent', color: '#ff6b6b', border: '1px solid #ff6b6b33', borderRadius: '6px', fontWeight: 600, cursor: 'pointer', fontSize: '13px' },
};
```

#### src/feedback/client-feedback.tsx

O componente principal le o estado `enabled` do store. Se `enabled === false`, retorna `null` (nada renderizado).

```tsx
import React, { useState, useCallback, useRef } from 'react';
import { Agentation, loadAnnotations, saveAnnotations } from 'agentation';
import { generateMarkdown, downloadMarkdown, copyMarkdownToClipboard } from './generate-markdown';
import { FeedbackPanel } from './feedback-panel';

// IMPORTANTE: importar o store criado no passo 2.
// Zustand: import { useFeedbackStore } from '../stores/feedbackStore';
// Sem Zustand: import { useFeedbackStore } from './use-feedback-store';

interface Annotation {
  id: string;
  comment: string;
  element: string;
  elementPath: string;
  x: number;
  y: number;
  timestamp: number;
  selectedText?: string;
  intent?: 'fix' | 'change' | 'question' | 'approve';
  severity?: 'blocking' | 'important' | 'suggestion';
}

interface ClientFeedbackProps {
  title?: string;
  includeMetadata?: boolean;
}

export function ClientFeedback({
  title = 'Client Feedback',
  includeMetadata = true,
}: ClientFeedbackProps) {
  // Le do store — retorna null se desativado
  const enabled = useFeedbackStore((s) => s.enabled);
  const [isActive, setIsActive] = useState(false);
  const [isPanelOpen, setIsPanelOpen] = useState(false);
  const [annotations, setAnnotations] = useState<Annotation[]>(() => {
    if (typeof window === 'undefined') return [];
    return loadAnnotations<Annotation>(window.location.pathname);
  });
  const [copyFeedback, setCopyFeedback] = useState(false);
  const [isSending, setIsSending] = useState(false);
  const [sendStatus, setSendStatus] = useState<'idle' | 'success' | 'error'>('idle');
  const copyTimeoutRef = useRef<ReturnType<typeof setTimeout>>(null);
  const sendTimeoutRef = useRef<ReturnType<typeof setTimeout>>(null);

  const persist = useCallback((anns: Annotation[]) => {
    saveAnnotations(window.location.pathname, anns);
  }, []);

  const handleAnnotationAdd = useCallback(
    (annotation: Annotation) => {
      setAnnotations((prev) => {
        const next = [...prev, annotation];
        persist(next);
        return next;
      });
    },
    [persist]
  );

  const handleAnnotationUpdate = useCallback(
    (annotation: Annotation) => {
      setAnnotations((prev) => {
        const next = prev.map((a) => (a.id === annotation.id ? annotation : a));
        persist(next);
        return next;
      });
    },
    [persist]
  );

  const handleAnnotationDelete = useCallback(
    (annotation: Annotation) => {
      setAnnotations((prev) => {
        const next = prev.filter((a) => a.id !== annotation.id);
        persist(next);
        return next;
      });
    },
    [persist]
  );

  const handleAnnotationsClear = useCallback(() => {
    setAnnotations([]);
    persist([]);
  }, [persist]);

  const handleDeleteFromPanel = useCallback(
    (id: string) => {
      setAnnotations((prev) => {
        const next = prev.filter((a) => a.id !== id);
        persist(next);
        return next;
      });
    },
    [persist]
  );

  const handleExport = useCallback(() => {
    downloadMarkdown(annotations, { title, includeMetadata });
  }, [annotations, title, includeMetadata]);

  const handleCopy = useCallback(async () => {
    await copyMarkdownToClipboard(annotations, { title, includeMetadata });
    setCopyFeedback(true);
    if (copyTimeoutRef.current) clearTimeout(copyTimeoutRef.current);
    copyTimeoutRef.current = setTimeout(() => setCopyFeedback(false), 2000);
  }, [annotations, title, includeMetadata]);

  const handleSend = useCallback(async () => {
    if (annotations.length === 0) return;

    setIsSending(true);
    setSendStatus('idle');

    try {
      const markdown = generateMarkdown(annotations, { title, includeMetadata });

      // Adaptar headers conforme o projeto (ex: CSRF token, auth cookies)
      const response = await fetch('/api/feedback', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        credentials: 'include',
        body: JSON.stringify({
          markdown,
          pageUrl: window.location.href,
          annotations,
        }),
      });

      if (response.ok) {
        setSendStatus('success');
        setAnnotations([]);
        persist([]);
        if (sendTimeoutRef.current) clearTimeout(sendTimeoutRef.current);
        sendTimeoutRef.current = setTimeout(() => setSendStatus('idle'), 3000);
      } else {
        setSendStatus('error');
      }
    } catch {
      setSendStatus('error');
    } finally {
      setIsSending(false);
    }
  }, [annotations, title, includeMetadata, persist]);

  const handleToggle = useCallback(() => {
    setIsActive((prev) => !prev);
  }, []);

  if (!enabled) return null;

  return (
    <>
      <button
        onClick={handleToggle}
        style={{ ...styles.toggleBtn, background: isActive ? '#7c83ff' : '#1a1a2e' }}
        title={isActive ? 'Desativar feedback' : 'Ativar feedback'}
      >
        <span style={styles.toggleIcon}>{isActive ? '\u25CF' : '\u25CB'}</span>
        <span>Feedback</span>
        {annotations.length > 0 && <span style={styles.badge}>{annotations.length}</span>}
      </button>

      {annotations.length > 0 && (
        <button
          onClick={() => setIsPanelOpen((prev) => !prev)}
          style={styles.panelToggleBtn}
          title="Ver anotacoes"
        >
          {isPanelOpen ? 'X' : `${annotations.length}`}
        </button>
      )}

      {isPanelOpen && (
        <FeedbackPanel
          annotations={annotations}
          onExport={handleExport}
          onCopy={handleCopy}
          onClear={handleAnnotationsClear}
          onClose={() => setIsPanelOpen(false)}
          onDeleteAnnotation={handleDeleteFromPanel}
          onSend={handleSend}
          isSending={isSending}
          sendStatus={sendStatus}
        />
      )}

      {copyFeedback && <div style={styles.toast}>Copiado para a area de transferencia!</div>}

      {isActive && (
        <Agentation
          copyToClipboard={false}
          onAnnotationAdd={handleAnnotationAdd}
          onAnnotationUpdate={handleAnnotationUpdate}
          onAnnotationDelete={handleAnnotationDelete}
          onAnnotationsClear={handleAnnotationsClear}
        />
      )}
    </>
  );
}

const styles: Record<string, React.CSSProperties> = {
  toggleBtn: { position: 'fixed', bottom: '20px', right: '20px', zIndex: 99998, display: 'flex', alignItems: 'center', gap: '8px', padding: '10px 18px', border: '1px solid #333', borderRadius: '50px', color: '#fff', fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif', fontSize: '14px', fontWeight: 600, cursor: 'pointer', boxShadow: '0 4px 20px rgba(0,0,0,0.3)', transition: 'all 0.2s ease' },
  toggleIcon: { fontSize: '10px' },
  badge: { background: '#ff6b6b', color: '#fff', borderRadius: '50%', width: '22px', height: '22px', display: 'flex', alignItems: 'center', justifyContent: 'center', fontSize: '11px', fontWeight: 700 },
  panelToggleBtn: { position: 'fixed', bottom: '80px', right: '20px', zIndex: 99998, padding: '8px 14px', background: '#1a1a2e', border: '1px solid #333', borderRadius: '50px', color: '#fff', fontSize: '13px', cursor: 'pointer', boxShadow: '0 4px 12px rgba(0,0,0,0.2)', fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif' },
  toast: { position: 'fixed', bottom: '80px', right: '100px', zIndex: 99999, background: '#2ecc71', color: '#fff', padding: '10px 20px', borderRadius: '8px', fontSize: '13px', fontWeight: 600, boxShadow: '0 4px 12px rgba(0,0,0,0.2)', fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif' },
};
```

#### src/feedback/index.ts

```typescript
export { ClientFeedback } from './client-feedback';
export { generateMarkdown, downloadMarkdown, copyMarkdownToClipboard } from './generate-markdown';
export { FeedbackPanel } from './feedback-panel';
```

### 4. Integrar toggle nas Configuracoes do projeto

**OBRIGATORIO:** Encontrar a pagina/modal de configuracoes do projeto e adicionar um toggle switch para ativar/desativar o modo feedback. O estado inicial DEVE ser desativado.

Explorar o projeto para encontrar:
- Settings modal/page (ex: `SettingsModal.tsx`, `Settings.tsx`, `settings/page.tsx`)
- Se o modal usa tabs, adicionar uma nova tab "Ferramentas" ou adicionar o toggle em uma tab existente
- Se nao existe pagina de configuracoes, criar uma secao minima

#### Padrao do Toggle Switch (CSS puro, sem dependencia)

O toggle segue o padrao de switch acessivel com `role="switch"` e `aria-checked`:

```tsx
// Importar o store
// Zustand: import { useFeedbackStore } from '../../stores/feedbackStore';
// Sem Zustand: import { useFeedbackStore } from '../../feedback/use-feedback-store';

// Dentro do componente de settings:
const feedbackEnabled = useFeedbackStore((s) => s.enabled);
const toggleFeedback = useFeedbackStore((s) => s.toggle);
// Sem Zustand:
// const { enabled: feedbackEnabled, toggle: toggleFeedback } = useFeedbackStore();
```

```tsx
{/* Toggle de Feedback — adicionar no componente de settings */}
<div className="border border-border rounded-xl p-5 bg-white/[0.02]">
  <div className="flex items-start justify-between">
    <div className="flex items-center gap-3">
      <div className="w-10 h-10 bg-indigo-500/10 rounded-xl flex items-center justify-center">
        {/* Icone — usar o que o projeto tiver (lucide, heroicons, svg inline, etc) */}
        <MessageCircleIcon className="w-5 h-5 text-indigo-400" />
      </div>
      <div>
        <h3 className="text-sm font-bold text-white">Modo Feedback</h3>
        <p className="text-[10px] text-muted-foreground mt-0.5">
          Permite que clientes cliquem em elementos e deixem comentarios visuais
        </p>
      </div>
    </div>
    <button
      type="button"
      role="switch"
      aria-checked={feedbackEnabled}
      onClick={toggleFeedback}
      className={`relative inline-flex h-6 w-11 items-center rounded-full transition-colors focus:outline-none flex-shrink-0 ${
        feedbackEnabled ? 'bg-indigo-500' : 'bg-white/10'
      }`}
    >
      <span
        className={`inline-block h-4 w-4 rounded-full bg-white transition-transform ${
          feedbackEnabled ? 'translate-x-6' : 'translate-x-1'
        }`}
      />
    </button>
  </div>
  {feedbackEnabled && (
    <div className="mt-4 pt-4 border-t border-border">
      <p className="text-xs text-muted-foreground leading-relaxed">
        Um botao "Feedback" aparecera no canto inferior direito da tela.
        Clique nele para ativar o modo de anotacao, depois clique em qualquer
        elemento da pagina para deixar um comentario.
      </p>
    </div>
  )}
</div>
```

**Adaptacoes por projeto:**
- Se o projeto usa Tailwind: usar as classes acima (ajustar cores conforme o tema)
- Se nao usa Tailwind: converter para CSS inline ou CSS modules
- Se o settings usa tabs: adicionar tab "Ferramentas" ou "Tools" e colocar o toggle la
- Se nao tem tabs: adicionar como uma secao no final das configuracoes
- Adaptar icone ao sistema do projeto (lucide-react, heroicons, svg, etc)

### 5. Montar componente no layout raiz

Detectar o framework e adicionar o componente **SEM forceEnable** (o store controla a visibilidade):

**Next.js App Router** (`app/layout.tsx`):
```tsx
import { ClientFeedback } from '@/feedback';

// Adicionar dentro do body, apos {children}:
<ClientFeedback />
```

**Next.js Pages Router** (`pages/_app.tsx`):
```tsx
import { ClientFeedback } from '@/feedback';

// Adicionar apos <Component {...pageProps} />:
<ClientFeedback />
```

**Vite/React** (`src/App.tsx` ou `src/main.tsx`):
```tsx
import { ClientFeedback } from './feedback';

// Adicionar no componente raiz, proximo ao final do JSX:
<ClientFeedback />
```

### 6. Configurar variaveis de ambiente

Adicionar ao `.env` do projeto (ou `.env.local` em Next.js):

```env
# Suna Feedback Integration
SUNA_API_URL=https://suna-api.claudedokploy.com/api
SUNA_API_KEY=PUBLIC_KEY:SECRET_KEY
SUNA_PROJECT_ID=uuid-do-projeto-no-suna
```

**Como obter:**
- `SUNA_API_KEY`: Criar em Suna > Settings > API Keys (formato `public:secret`)
- `SUNA_PROJECT_ID`: ID do projeto no Suna (UUID). Copiar da URL do projeto no dashboard

### 7. Criar API route para envio ao Suna

A API route recebe o feedback do widget e encaminha pro Suna como nota vinculada ao projeto.

#### Next.js App Router (`src/app/api/feedback/route.ts`)

```typescript
import { NextRequest, NextResponse } from 'next/server';

const SUNA_API_URL = process.env.SUNA_API_URL;
const SUNA_API_KEY = process.env.SUNA_API_KEY;
const SUNA_PROJECT_ID = process.env.SUNA_PROJECT_ID;

export async function POST(request: NextRequest) {
  try {
    const { markdown, pageUrl, annotations } = await request.json();

    if (!markdown) {
      return NextResponse.json({ error: 'Markdown content is required' }, { status: 400 });
    }

    // Se Suna estiver configurado, enviar como nota vinculada ao projeto
    if (SUNA_API_URL && SUNA_API_KEY && SUNA_PROJECT_ID) {
      const sunaResponse = await fetch(`${SUNA_API_URL}/notes/capture`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'x-api-key': SUNA_API_KEY,
        },
        body: JSON.stringify({
          title: `Feedback: ${pageUrl}`,
          content: markdown,
          project_id: SUNA_PROJECT_ID,
          note_type: 'general',
        }),
      });

      if (!sunaResponse.ok) {
        const err = await sunaResponse.text();
        console.error('[Feedback] Suna API error:', sunaResponse.status, err);
        return NextResponse.json(
          { error: 'Failed to send feedback to project' },
          { status: 502 }
        );
      }

      const note = await sunaResponse.json();
      return NextResponse.json({
        success: true,
        noteId: note.id,
        message: 'Feedback enviado e vinculado ao projeto',
      });
    }

    // Fallback: logar localmente se Suna nao estiver configurado
    console.log('[Feedback]', { pageUrl, count: annotations?.length });
    return NextResponse.json({ success: true });
  } catch (error) {
    console.error('Feedback API error:', error);
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 });
  }
}
```

#### Express / Hono (Node.js)

```typescript
import { Router } from 'express';

const router = Router();

router.post('/api/feedback', async (req, res) => {
  try {
    const { markdown, pageUrl, annotations } = req.body;

    if (!markdown) {
      return res.status(400).json({ error: 'Markdown content is required' });
    }

    const { SUNA_API_URL, SUNA_API_KEY, SUNA_PROJECT_ID } = process.env;

    if (SUNA_API_URL && SUNA_API_KEY && SUNA_PROJECT_ID) {
      const sunaResponse = await fetch(`${SUNA_API_URL}/notes/capture`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'x-api-key': SUNA_API_KEY,
        },
        body: JSON.stringify({
          title: `Feedback: ${pageUrl}`,
          content: markdown,
          project_id: SUNA_PROJECT_ID,
          note_type: 'general',
        }),
      });

      if (!sunaResponse.ok) {
        const err = await sunaResponse.text();
        console.error('[Feedback] Suna API error:', sunaResponse.status, err);
        return res.status(502).json({ error: 'Failed to send feedback to project' });
      }

      const note = await sunaResponse.json();
      return res.json({
        success: true,
        noteId: note.id,
        message: 'Feedback enviado e vinculado ao projeto',
      });
    }

    // Fallback
    console.log('[Feedback]', { pageUrl, count: annotations?.length });
    return res.json({ success: true });
  } catch (error) {
    console.error('[Feedback] Error:', error);
    return res.status(500).json({ error: 'Internal server error' });
  }
});

export function registerFeedbackRoutes(app) {
  app.use(router);
}
```

Lembrar de:
- Registrar a rota no app principal (ex: `registerFeedbackRoutes(app)`)
- Adicionar `/api/feedback` nas rotas protegidas se o projeto usar auth/CSRF
- **Nunca expor** `SUNA_API_KEY` no client-side (manter so em server env)

## Fluxo do Usuario

1. Usuario vai em **Configuracoes > Ferramentas** e ativa o toggle "Modo Feedback"
2. Botao "Feedback" aparece no canto inferior direito
3. Clica para ativar modo de anotacao (botao fica roxo)
4. Clica em elementos da pagina e deixa comentarios
5. Abre painel lateral para ver/gerenciar anotacoes
6. Clica "Enviar" → feedback vai pro Suna como nota vinculada ao projeto
7. Desativa o toggle em Configuracoes para esconder o botao

## Fluxo Tecnico (Envio)

```
Widget (client-side) → POST /api/feedback (server route do app)
  → POST SUNA_API_URL/notes/capture (server-to-server)
    → Nota criada no Suna, vinculada ao SUNA_PROJECT_ID
```

- A API key fica **server-side** (nunca exposta ao browser)
- O `project_id` eh fixo por app/projeto — toda nota de feedback chega vinculada
- O Suna converte o markdown pra formato Plate automaticamente

## Ativacao

**Metodo principal:** Toggle em Configuracoes (persiste em localStorage via store)

- Estado inicial: **DESATIVADO**
- Usuario ativa/desativa quando quiser
- Nao aparece nada no front-end ate o usuario ativar

## Integracao Suna

Para que o feedback chegue como nota no Suna vinculada a um projeto:

1. **Criar API Key no Suna:** Settings > API Keys > Criar nova key
2. **Copiar o Project ID:** URL do projeto no Suna (UUID)
3. **Configurar .env** com `SUNA_API_URL`, `SUNA_API_KEY`, `SUNA_PROJECT_ID`
4. A API route faz o forward automaticamente

Se as env vars nao estiverem configuradas, o feedback eh logado localmente como fallback.

## Notes

- Anotacoes persistem em localStorage por pagina
- O componente Agentation so renderiza quando o modo de anotacao esta ativo
- Estilos sao inline (zero dependencia de CSS framework)
- Toggle switch usa CSS puro com Tailwind (adaptar se projeto nao usa Tailwind)
- Compativel com React 18+ e Next.js 13+
- Se o projeto tiver CSRF protection, adaptar o fetch no `handleSend` para incluir o token
- Endpoint Suna: `POST /notes/capture` aceita `{ title, content (markdown), project_id, note_type }`
- Auth Suna: header `x-api-key` com formato `public_key:secret_key`
