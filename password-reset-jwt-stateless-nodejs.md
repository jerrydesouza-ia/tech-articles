# Token JWT Auto-Invalidador para Reset de Senha — Técnica Stateless

> Implementação em **Next.js / Node.js** sem criar tabelas extras, sem cron jobs e com invalidação automática ao ser usado.

## O Problema Clássico

A maioria dos sistemas de recuperação de senha funciona assim:

1. Gerar um token aleatório
2. Salvar esse token em uma tabela `password_reset_tokens` no banco de dados
3. Enviar por e-mail com prazo de expiração
4. Ao usar, marcar como `used: true` ou deletar o registro

**Problemas dessa abordagem:**
- Exige migration de banco de dados
- Exige limpeza periódica (cron job) para remover tokens expirados
- Aumenta a complexidade da modelagem
- Mais queries no banco por operação

---

## A Solução: Assinatura Dinâmica com `passwordHash`

A técnica utiliza o próprio **hash bcrypt da senha atual** do usuário como parte do segredo do JWT:

```
secret = JWT_SECRET + user.passwordHash
token  = jwt.sign({ userId }, secret, { expiresIn: '30m' })
```

### Por que isso é seguro e elegante?

| Propriedade | Como é garantida |
|---|---|
| **Expiração** | `expiresIn: '30m'` nativo do JWT |
| **Invalidação por uso** | Ao redefinir a senha, o `passwordHash` muda → o `secret` muda → o token antigo se torna inválido |
| **Invalidação por mudança de senha** | Mesma razão acima — qualquer alteração de senha invalida todos os tokens pendentes |
| **Zero estado no banco** | Nenhuma tabela criada, nenhuma linha gravada |

---

## Implementação Completa

### 1. Gerar o token (`forgot-password`)

```typescript
// src/app/api/auth/forgot-password/route.ts
import jwt from 'jsonwebtoken';
import { prisma } from '@/lib/db';
import { sendResetPasswordEmail } from '@/lib/mail';

const JWT_SECRET_BASE = process.env.JWT_SECRET!;

export async function POST(request: Request) {
    const { identifier } = await request.json();

    const user = await prisma.user.findFirst({
        where: {
            OR: [
                { email: identifier.toLowerCase() },
                { cpf: identifier.replace(/\D/g, '') }
            ]
        }
    });

    // Resposta sempre genérica — evita user enumeration (OWASP)
    const successMsg = {
        message: 'Se o cadastro existir, um link de recuperação será enviado.'
    };

    if (!user || !user.passwordHash) {
        return Response.json(successMsg);
    }

    // Chave dinâmica: JWT_SECRET + hash atual da senha
    const secret = JWT_SECRET_BASE + user.passwordHash;
    const token = jwt.sign({ userId: user.id }, secret, { expiresIn: '30m' });

    // Fire-and-forget para não travar a resposta se o SMTP falhar
    sendResetPasswordEmail(user.email, user.name, token).catch(console.error);

    return Response.json(successMsg);
}
```

### 2. Validar e redefinir (`reset-password`)

```typescript
// src/app/api/auth/reset-password/route.ts
import jwt from 'jsonwebtoken';
import bcrypt from 'bcryptjs';
import { prisma } from '@/lib/db';

const JWT_SECRET_BASE = process.env.JWT_SECRET!;

export async function POST(request: Request) {
    const { token, newPassword } = await request.json();

    // Decodifica sem verificar — apenas para pegar o userId
    const decoded = jwt.decode(token) as { userId: string } | null;
    if (!decoded?.userId) {
        return Response.json({ error: 'Token inválido.' }, { status: 400 });
    }

    const user = await prisma.user.findUnique({ where: { id: decoded.userId } });
    if (!user?.passwordHash) {
        return Response.json({ error: 'Usuário inválido.' }, { status: 400 });
    }

    // Verifica a assinatura com o hash atual — se a senha já mudou, falha aqui
    try {
        jwt.verify(token, JWT_SECRET_BASE + user.passwordHash);
    } catch {
        return Response.json(
            { error: 'Link expirado ou já utilizado.' },
            { status: 400 }
        );
    }

    // Atualiza a senha — o hash muda, invalidando o token automaticamente
    const newHash = await bcrypt.hash(newPassword, 10);
    await prisma.user.update({
        where: { id: user.id },
        data: { passwordHash: newHash }
    });

    return Response.json({ message: 'Senha redefinida com sucesso!' });
}
```

---

## Teste Unitário (Vitest)

```typescript
import { describe, it, expect } from 'vitest';
import jwt from 'jsonwebtoken';
import bcrypt from 'bcryptjs';

describe('Token JWT auto-invalidador', () => {
    it('invalida o token ao mudar o passwordHash', async () => {
        const hash1 = await bcrypt.hash('senhaAntiga', 10);
        const secret = 'jwt-base-secret';

        const token = jwt.sign({ userId: '123' }, secret + hash1, { expiresIn: '30m' });

        // Token válido com hash original
        expect(() => jwt.verify(token, secret + hash1)).not.toThrow();

        // Simula mudança de senha
        const hash2 = await bcrypt.hash('senhaNova', 10);

        // Token inválido com novo hash — invalidação automática ✅
        expect(() => jwt.verify(token, secret + hash2)).toThrow();
    });
});
```

---

## Fluxo Visual

```
[Usuário solicita reset]
        │
        ▼
[Backend busca usuário no banco]
        │
        ▼
[Assina JWT com: secret = JWT_SECRET + passwordHash_atual]
        │
        ▼
[Envia link por e-mail com token]
        │
        ▼
[Usuário clica no link → POST /reset-password]
        │
        ▼
[Backend busca usuário, pega passwordHash_atual do banco]
        │
        ▼
[Verifica JWT com: secret = JWT_SECRET + passwordHash_atual]
     /      \
  Válido   Inválido (expirou ou já foi usado)
    │
    ▼
[Atualiza passwordHash → token automaticamente inválido]
```

---

## Por que Funciona?

Ao redefinir a senha:

```
passwordHash_antes: $2b$10$abc123...
passwordHash_depois: $2b$10$xyz789...  ← completamente diferente
```

Como o `secret` do JWT incluía o hash antigo, qualquer tentativa de verificar o mesmo token com o hash novo **falhará com `JsonWebTokenError: invalid signature`** — sem qualquer escrita adicional no banco.

---

## Stack utilizada

- **Next.js** (App Router / Route Handlers)
- **Prisma ORM** + **PostgreSQL**
- **jsonwebtoken** (`jwt.sign` / `jwt.verify`)
- **bcryptjs** (hash de senhas)
- **Nodemailer** (envio de e-mails)
- **Vitest** (testes unitários)

---

## Créditos

Implementado no projeto [ASFControl](https://asfcontrol.com.br) — plataforma de gestão e rastreabilidade para meliponicultura brasileira.
