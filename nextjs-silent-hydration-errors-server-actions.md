# Quando o Next.js falha em silêncio: Como erros de Hidratação "matam" seus botões e Server Actions

Se você trabalha com Next.js (especialmente com o App Router), provavelmente já passou por isso: você cria um formulário ou um fluxo de onboarding lindo, escreve seu *Client Component*, adiciona um `onClick` chamando uma *Server Action*... e ao testar em produção (ou até no `npm run dev`), você clica no botão e **absolutamente nada acontece**.

Sem erros na tela. Sem `console.log`. Apenas um botão morto.

Recentemente, enfrentei exatamente esse cenário enquanto desenvolvia um fluxo crítico de Onboarding no **ASFControl**. O problema me fez revisitar os conceitos de hidratação do React e me lembrou de uma lição valiosa: às vezes, a melhor solução no desenvolvimento web moderno é voltar ao básico.

Neste artigo, vou explicar por que isso acontece e como resolver (ou contornar) de forma à prova de falhas.

---

## O Sintoma: O "Clique Fantasma"

O cenário era simples: um botão de "Concluir Onboarding" que deveria:
1. Chamar uma função assíncrona para atualizar o banco de dados.
2. Redirecionar o usuário para a rota `/dashboard`.

O código era mais ou menos assim:

```tsx
"use client";
import { Button } from "@/components/ui/button";
import { markOnboardingComplete } from "@/actions/onboarding";

export function SetupClient() {
    const handleFinish = async () => {
        setIsLoading(true);
        try {
            await markOnboardingComplete(); // Server Action
        } catch (error) {}
        
        window.location.href = "/dashboard";
    };

    return (
        <Button onClick={handleFinish}>
            Ir para a Dashboard
        </Button>
    );
}
```

Visualmente, a página carregava perfeitamente. Porém, ao clicar, nada acontecia. O `isLoading` não mudava, a requisição não saía no painel *Network*, e o redirecionamento não ocorria.

## O Culpado: A Falha Silenciosa de Hidratação

No Next.js, a página é inicialmente renderizada no servidor (SSR) e enviada como HTML puro para o navegador. Em seguida, o React entra em ação (um processo chamado **Hidratação**) para anexar os eventos de Javascript (como o nosso `onClick`) aos elementos HTML.

Se houver **qualquer** inconsistência entre o HTML gerado pelo servidor e o HTML esperado pelo cliente — como uma tag `<div>` fechada no lugar errado, ou uma extensão do Chrome injetando código no DOM antes da hora — o React falha ao hidratar aquela árvore de componentes.

O pior de tudo? Em muitos casos, **o React simplesmente desiste de anexar os eventos àquele pedaço do DOM e não avisa o usuário**. O botão fica na tela (pois é HTML), mas o "cérebro" Javascript dele nunca é conectado. A Server Action fica inacessível.

## A Solução Robusta em 2 Passos

Quando você está lidando com fluxos críticos (login, checkout, onboarding), você não pode depender de que 100% dos usuários terão uma hidratação perfeita. Você precisa criar rotas de escape.

### 1. Troque Server Actions por API Routes para chamadas críticas no Client

Server Actions são excelentes, mas em componentes muito dinâmicos sujeitos a falhas de renderização, elas podem ser frágeis (especialmente se houver mismatch de cache). Voltar ao bom e velho `fetch` para uma API Route dedicada (`/api/...`) garante um comportamento muito mais previsível.

### 2. O Padrão "Fallback de Navegação Nativa"

A solução definitiva que aplicamos no ASFControl foi **remover a dependência do Javascript para a navegação principal**. Se o botão tem como objetivo principal levar o usuário para o `/dashboard`, por que usar um evento `onClick` artificial? 

Nós transformamos o botão em uma tag HTML `<a>` pura, e movemos a atualização de estado para o **Layout do Servidor** da página de destino.

```tsx
// O Client Component (SetupClient.tsx) vira apenas:
<div className="pt-6">
    <a 
        href="/dashboard"
        className="w-full bg-black hover:bg-gray-800 text-white font-bold h-12 flex items-center justify-center rounded-md"
    >
        Ir para a Dashboard
    </a>
</div>
```

Zero Javascript. Zero chance de falha por hidratação. O clique sempre vai funcionar.

**"Mas e a atualização do banco de dados?"**

Ao invés de tentar fazer a atualização no evento de clique do botão, movemos essa inteligência para o `layout.tsx` da página `/dashboard`:

```tsx
// dashboard/layout.tsx (Server Component)
export default async function DashboardLayout({ children }) {
    const user = await getUserData();
    
    // Auto-correção (Self-Healing)
    if (user._count.meliponarios > 0 && !user.onboardingCompleted) {
        await markOnboardingComplete(user.id);
    }
    
    // ...
}
```

### O Redirecionamento Automático (A "Cereja do Bolo")

Para melhorar ainda mais a UX e evitar que o usuário precise clicar no link, adicionamos um redirecionamento automático (sempre mantendo o link `<a href>` visível como fallback à prova de balas caso o JS falhe):

```tsx
// Auto-redirecionamento quando chega no último passo
useEffect(() => {
    if (step === 3) {
        // Tenta notificar o backend assincronamente (sem bloquear a navegação)
        fetch('/api/auth/onboarding-complete', { method: 'POST' }).catch(() => {});
        
        // Navega automaticamente após 2.5 segundos
        const timer = setTimeout(() => {
            window.location.href = "/dashboard";
        }, 2500);
        return () => clearTimeout(timer);
    }
}, [step]);
```

## Conclusão

O ecossistema moderno do React nos dá ferramentas incríveis, mas a complexidade traz novos tipos de falhas silenciosas. O "Clique Fantasma" causado por erros de hidratação é um lembrete de que:

1. **Nem sempre a culpa é da sua lógica.** A infraestrutura por trás do framework pode estar falhando.
2. **HTML puro nunca falha.** Sempre que possível, use âncoras `<a>` e formulários `<form>` nativos para ações críticas de navegação e mutação.
3. **Sistemas Resilientes (Self-Healing).** Desenhe suas rotas de destino para corrigir o estado do banco de dados automaticamente, não dependa apenas do sucesso da requisição que originou o redirecionamento.

Como costumamos dizer na arquitetura do ASFControl: a interface de usuário deve ser fluida, mas a engenharia por trás dela deve ser implacável.

---
*Gostou do artigo? Confira o projeto ASFControl e mais conteúdos sobre arquitetura de software no meu [GitHub](https://github.com/jerrydesouza-ia).*
