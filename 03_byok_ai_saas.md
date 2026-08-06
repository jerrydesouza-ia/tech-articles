# Escalando um SaaS com Inteligência Artificial sem quebrar o banco: A Estratégia BYOK

Como fundadores e desenvolvedores de SaaS, a integração de Inteligência Artificial deixou de ser um diferencial e se tornou uma exigência. Contudo, embutir o consumo de LLMs (como Google Gemini ou OpenAI) diretamente na precificação do seu software cria um problema clássico: **custos variáveis imprevisíveis**.

Recentemente, enfrentei esse exato desafio na arquitetura do **ASFControl**, nossa plataforma de gestão e rastreabilidade para meliponicultura. Como poderíamos oferecer análises avançadas de manejo, cruzamento de dados ambientais e insights genéticos via IA sem expor a plataforma a contas astronômicas de API?

A resposta foi adotar a arquitetura **BYOK (Bring Your Own Key)**.

Neste artigo, vamos explorar por que essa abordagem protege sua margem de lucro e como nós a implementamos tecnicamente.

---

## O Dilema dos Custos Variáveis na IA

O modelo tradicional de SaaS opera sob a premissa de custos marginais decrescentes: hospedar um usuário adicional custa frações de centavos no banco de dados (especialmente usando arquiteturas serverless e otimizações pesadas).

Porém, quando você introduz LLMs na equação, a conta muda. Cada prompt executado por um usuário consome tokens. Um usuário muito engajado (o famoso *heavy user*) pode rapidamente consumir mais valor em chamadas de API do que ele paga na assinatura mensal. 

Para piorar, você precisa gerenciar sistemas complexos de *Rate Limiting* (controle de limite de requisições) para evitar abusos e monitorar o uso da sua chave central constantemente.

## A Solução: Arquitetura BYOK (Bring Your Own Key)

BYOK significa "Traga sua própria chave". Em vez do SaaS pagar pelo processamento da IA, nós construímos a **Interface, o Workflow e a Engenharia de Prompts (Prompts Validados)**. O processamento (e o custo da API), no entanto, fica na conta do próprio usuário.

### O Case do ASFControl

No ecossistema do ASFControl, estabelecemos uma separação arquitetural estrita:

1. **Plano Guardião (FREE):** O produtor tem acesso total a toda a gestão operacional (colmeias, vendas, floradas, rastreabilidade).
2. **Plano ASFControl IA (Pago):** O cliente não está "comprando créditos de IA". Ele paga uma mensalidade fixa pelo acesso ao **Hub de Inteligência** e pela interface especializada de análise.
3. **A Chave BYOK:** Para que a inteligência funcione, o usuário prêmio gera a sua própria chave gratuita no Google AI Studio (Gemini) e a salva na página de Configurações da plataforma.

A chave de ambiente (`GEMINI_API_KEY`) registrada no `.env` do nosso servidor (Next.js/Node) fica restrita apenas aos administradores (`userType === 'ADMIN'`) para testes e debugging. Nenhum usuário final consome a cota da plataforma.

## Como Implementar na Prática (Next.js / Node.js)

Do ponto de vista técnico, você não deve expor a chave do cliente no client-side (`localStorage` + chamadas diretas pelo navegador). Isso é inseguro e bloqueado pelas políticas de CORS de muitos provedores de IA.

A abordagem ideal é armazenar a chave criptografada no banco de dados e descriptografá-la no backend para fazer as requisições em nome do usuário.

```typescript
// Exemplo de Handler no Next.js (Server Action ou API Route)
import { verifyToken } from "@/lib/auth";
import { GoogleGenerativeAI } from "@google/generative-ai";
import prisma from "@/lib/prisma";

export async function generateAIInsight(prompt: string, userToken: string) {
  // 1. Validar usuário logado
  const decoded = verifyToken(userToken);
  if (!decoded) throw new Error("Não autorizado");

  // 2. Buscar a chave individual do usuário no banco (Prisma)
  const user = await prisma.user.findUnique({
    where: { id: decoded.userId },
    select: { apiKeyGemini: true, role: true }
  });

  // 3. Estratégia de Fallback (Admin usa a chave do sistema)
  const apiKey = user.role === 'ADMIN' 
    ? process.env.GEMINI_API_KEY 
    : user.apiKeyGemini;

  if (!apiKey) {
    throw new Error("Chave de API não configurada. Acesse as configurações para inserir sua chave do Gemini.");
  }

  // 4. Instanciar o cliente do LLM usando a chave dinâmica
  const genAI = new GoogleGenerativeAI(apiKey);
  const model = genAI.getGenerativeModel({ model: "gemini-1.5-flash" });

  // 5. Executar o prompt pré-construído (A verdadeira inteligência do seu app)
  const result = await model.generateContent(prompt);
  return result.response.text();
}
```

### O que acontece com o Rate Limiting?

A grande vantagem técnica do BYOK é que **nós delegamos o Rate Limiting para o provedor do LLM**. Como cada cliente está utilizando uma chave diferente, o ASFControl não precisa de sistemas complexos de filas ou *Redis Throttling* para as chamadas de inteligência. Se o usuário abusar das requisições, ele baterá no limite da sua própria conta do Google AI Studio, deixando o restante da plataforma intacto.

## Conclusão

Sistemas BYOK resolvem três grandes problemas de arquiteturas de software modernas com IA:
1. **Margem Previsível:** Você cobra pelo valor da sua interface, integração e prompts, sem se preocupar com custos variáveis invisíveis.
2. **Zero Overhead de Servidor:** Você não precisa de sistemas gigantescos de controle de créditos e rate limiting.
3. **Transparência e Privacidade:** O usuário tem controle sobre sua própria chave, podendo revogá-la a qualquer momento em seu próprio painel do Google.

Se você está construindo um SaaS com ferramentas de IA avançadas em nichos específicos (B2B ou B2C profissional), a estratégia BYOK deve estar no topo das suas decisões de arquitetura.

---
**Gostou do artigo?** Vamos nos conectar para trocar mais experiências sobre desenvolvimento, SaaS e arquitetura de software!

🔗 **LinkedIn:** [https://www.linkedin.com/in/jerrydesouza-ia/](https://www.linkedin.com/in/jerrydesouza-ia/)
🐙 **GitHub:** [https://github.com/jerrydesouza-ia](https://github.com/jerrydesouza-ia)
