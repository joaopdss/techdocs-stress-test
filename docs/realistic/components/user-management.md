# User Management

## Descrição

O módulo User Management é a interface administrativa para gestão de usuários dentro do Admin Dashboard. Este módulo permite que operadores de suporte e administradores visualizem, editem e gerenciem contas de usuários da plataforma TechCorp.

O módulo oferece funcionalidades completas de gestão de ciclo de vida do usuário, desde a visualização de dados cadastrais até operações avançadas como merge de contas duplicadas, bloqueio por fraude e gerenciamento de permissões.

Diferente do user-service que é o backend de dados, este módulo é focado na experiência do operador administrativo, oferecendo workflows otimizados para as tarefas mais comuns de suporte ao cliente.

## Responsáveis

- **Time:** Internal Tools
- **Tech Lead:** Diego Martins
- **Slack:** #internal-admin

## Acesso

O módulo está disponível em: `https://admin.techcorp.com/users`

### Permissões Necessárias

| Ação | Perfil Mínimo |
|------|---------------|
| Visualizar usuário | support |
| Editar dados cadastrais | operations |
| Bloquear/Desbloquear | operations |
| Resetar senha | support |
| Merge de contas | admin |
| Excluir usuário | admin |

## Funcionalidades

### 1. Busca de Usuários

A busca permite encontrar usuários por múltiplos critérios:

| Campo | Tipo de Busca |
|-------|---------------|
| E-mail | Exata ou parcial |
| CPF | Exata (com ou sem formatação) |
| Telefone | Exata ou últimos 4 dígitos |
| Nome | Parcial (fuzzy) |
| ID do pedido | Busca o usuário associado |

### 2. Visualização de Perfil

O perfil do usuário exibe:

- **Dados Cadastrais:** Nome, e-mail, telefone, CPF, data de nascimento
- **Status:** Ativo, Inativo, Bloqueado, Pendente de verificação
- **Histórico de Pedidos:** Últimos 10 pedidos com link para detalhes
- **Endereços:** Lista de endereços cadastrados
- **Preferências:** Configurações de notificação e idioma
- **Timeline:** Histórico de ações na conta

### 3. Edição de Dados

Campos editáveis pelo operador:

| Campo | Validação |
|-------|-----------|
| Nome | Mínimo 2 caracteres |
| E-mail | Formato válido, único no sistema |
| Telefone | Formato válido (+55...) |
| Data de nascimento | Data válida, maior de 18 anos |

**Nota:** Alteração de e-mail requer confirmação do usuário via link enviado ao novo endereço.

### 4. Bloqueio de Conta

Motivos disponíveis para bloqueio:

- Suspeita de fraude
- Solicitação do usuário
- Inadimplência
- Violação de termos
- Outros (requer descrição)

### 5. Reset de Senha

O operador pode solicitar reset de senha, que envia um link de recuperação para o e-mail do usuário. O operador **não** tem acesso à senha atual nem pode definir uma nova senha diretamente.

### 6. Merge de Contas

Processo para unificar contas duplicadas:

1. Selecionar conta principal (que será mantida)
2. Selecionar conta secundária (que será desativada)
3. Revisar dados que serão migrados:
   - Pedidos
   - Endereços
   - Preferências
   - Saldo de créditos
4. Confirmar operação (irreversível)

## Workflows Comuns

### Cliente não consegue acessar a conta

1. Buscar usuário por e-mail ou CPF
2. Verificar status da conta
3. Se bloqueada, verificar motivo na timeline
4. Se ativa, solicitar reset de senha
5. Verificar se e-mail está correto

### Suspeita de fraude

1. Buscar usuário
2. Analisar histórico de pedidos
3. Verificar padrões suspeitos:
   - Múltiplos endereços de entrega
   - Alterações frequentes de dados
   - Chargebacks anteriores
4. Se confirmada suspeita, bloquear conta com motivo "Suspeita de fraude"
5. Notificar time de antifraude

### Solicitação LGPD (exclusão de dados)

1. Buscar usuário
2. Verificar se há pedidos em andamento
3. Se sim, aguardar conclusão
4. Exportar dados do usuário
5. Solicitar exclusão ao admin
6. Confirmar exclusão após período de retenção

## Troubleshooting

### Problema: Busca não encontra usuário que existe

**Causa:** Dados de busca inconsistentes ou cache desatualizado.

**Solução:**
1. Tentar busca por campo diferente (e-mail, CPF, telefone)
2. Verificar se CPF está formatado corretamente
3. Limpar filtros e tentar novamente
4. Usar ID do usuário se disponível

### Problema: Não consegue editar dados do usuário

**Causa:** Permissão insuficiente ou conta em estado especial.

**Solução:**
1. Verificar seu perfil de acesso
2. Verificar se conta não está em processo de exclusão
3. Verificar se há operação pendente (merge, verificação)
4. Solicitar elevação de permissão se necessário

### Problema: E-mail de reset não está chegando

**Causa:** E-mail em spam, bloqueado ou bounce.

**Solução:**
1. Verificar se e-mail cadastrado está correto
2. Solicitar que cliente verifique pasta de spam
3. Verificar status de entrega no notification-service
4. Se bounce, solicitar atualização do e-mail

### Problema: Merge de contas falhou

**Causa:** Conflito de dados ou erro no processamento.

**Solução:**
1. Verificar logs de erro no painel
2. Verificar se ambas as contas existem
3. Verificar se não há operação pendente em nenhuma conta
4. Tentar novamente ou escalar para suporte técnico

## Auditoria

Todas as ações no módulo são registradas:

- Usuário que executou a ação
- Data e hora
- Tipo de ação
- Dados antes e depois (para edições)
- IP de origem

Para consultar o histórico de auditoria, acesse a aba "Auditoria" no perfil do usuário ou use o módulo de Logs.

## Links Relacionados

- [User Service](user-service.md) - Backend de dados de usuário
- [Auth Service](auth-service.md) - Autenticação e reset de senha
- [Admin Dashboard](admin-dashboard.md) - Painel administrativo completo
- [API de Usuários](../apis/users-api.md) - Rotas da API
