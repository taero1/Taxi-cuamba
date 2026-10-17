API do App de Transportes (táxi, moto-táxi e carrinha)
Backend em Node.js + Express + PostgreSQL, feito para o schema.sql, o fluxo-telas.md e o plano-lancamento.md. Pensado para a realidade local: pagamento em dinheiro por defeito, código de acesso por SMS (sem senhas), pontos de referência além do GPS, limite de dívida de comissões e partilha da viagem sem instalar nada.
Estado desta versão (leia primeiro)
O quê
Estado
Migrações SQL (3 ficheiros)
Testadas num PostgreSQL 16 real: aplicam-se sem erros
Consultas SQL mais delicadas (distâncias, início do dia/semana/mês no fuso de Maputo, restrições "uma corrida ativa", cálculo de comissões)
Testadas no PostgreSQL 16 (isto apanhou um erro real no semear.js, já corrigido)
Sintaxe de todo o código JavaScript
Verificada (node --check)
Arranque da API e rotas HTTP
Não executadas nesta sessão: o ambiente onde o código foi escrito bloqueia o registo de pacotes do npm, por isso não foi possível instalar o Express nem correr o servidor
Teste de ponta a ponta (npm test)
Escrito, mas ainda não executado — é o primeiro passo que deve fazer (ver abaixo)
SMS
Só modo consola (o código aparece no terminal). Falta ligar um fornecedor real
M-Pesa / e-Mola
Não integrados: a API recusa esses métodos de propósito. Só dinheiro
Notificações em tempo real
Não há. A app deve consultar GET /api/corridas/atual e GET /api/motorista/pedidos a cada 3–5 segundos
Como o código das rotas nunca correu, é provável que apareça algum erro pequeno na primeira execução. O teste npm test percorre uma corrida completa e aponta onde falha.
Instalação
Precisa de Node.js 20 ou superior e PostgreSQL 14 ou superior.
npm install
cp .env.example .env          # edite DATABASE_URL e JWT_SEGREDO
createdb transportes_teste    # base só para testes
1. Correr o teste primeiro (base de testes)
No .env, aponte DATABASE_URL para .../transportes_teste e:
npm run migrar
npm test
O teste recusa-se a correr se o nome da base de dados não tiver "teste" ou "test".
2. Preparar a base real
createdb transportes
# no .env: DATABASE_URL=postgres://.../transportes
npm run migrar
npm run semear                                  # opcional: tarifas e pontos de EXEMPLO (edite!)
npm run criar-admin -- "84 123 4567" "O seu nome"
npm start
Para entrar como administrador: POST /api/auth/pedir-codigo com o seu número, veja o código no terminal e confirme em POST /api/auth/verificar-codigo.
Como a autenticação funciona
POST /api/auth/pedir-codigo { "telefone": "84 123 4567" } → SMS com código de 6 dígitos (válido 10 min, máx. 5 pedidos por hora, 5 tentativas).
POST /api/auth/verificar-codigo { "telefone", "codigo" } → devolve token. No primeiro acesso envie também "nome" e, para motoristas, "tipo": "motorista".
Em todos os outros pedidos: cabeçalho Authorization: Bearer <token>.
Contas suspensas ou bloqueadas perdem o acesso de imediato (o estado é lido da base de dados a cada pedido).
Rotas principais
Públicas: GET /saude · GET /api/zonas · GET /api/pontos?q=mercado · GET /api/partilha/:codigo (viagem partilhada, sem login)
Cliente
POST /api/corridas/estimativa — preço sem pedir
POST /api/corridas — pedir (tipo_veiculo, origem, destino; cada local é { "ponto_id": 3 } ou { "lat": -14.8, "lng": 36.5, "descricao": "..." })
GET /api/corridas/atual · GET /api/corridas/historico · GET /api/corridas/:id
POST /api/corridas/:id/cancelar · /avaliar · /emergencia
POST /api/reclamacoes · GET /api/reclamacoes
Motorista
POST /api/motorista/perfil (carta + veículo; fica "pendente" até a administração aprovar)
PUT /api/motorista/perfil · PUT /api/motorista/veiculo
POST /api/motorista/estado (online/offline) · POST /api/motorista/localizacao
GET /api/motorista/pedidos — pedidos abertos num raio de RAIO_PEDIDOS_KM
POST /api/corridas/:id/aceitar → /a-caminho → /iniciar → /concluir (e /desistir)
GET /api/motorista/ganhos?periodo=dia|semana|mes
Administração (/api/admin/...): dashboard, mapa, motoristas, motoristas/:id/verificacao, utilizadores/:id/estado, veiculos, documentos-a-expirar, veiculos/:id/manutencoes, corridas, corridas/:id/ajustar-preco, tarifas, zonas, pontos, comissoes, comissoes/pagamento, relatorio?de=&ate=&formato=csv (abre bem no Excel), reclamacoes, emergencias.
Regras de negócio (todas ajustáveis no .env)
Preço: tarifa fixa entre zonas, se existir; senão taxa_base + preço_por_km × distância, arredondado para cima a múltiplos de ARREDONDAR_PARA. A distância é em linha reta × FATOR_ROTA (aproximação; não usa ruas reais).
Preço final pode ultrapassar o estimado até AJUSTE_PRECO_MAX (30%). Disputas: a administração corrige em ajustar-preco (fica registado em ajustes_preco).
Comissão: COMISSAO_PERCENTAGEM do preço final, somada ao saldo do motorista. Acima de LIMITE_DIVIDA o motorista deixa de poder ficar online ou aceitar pedidos até pagar. A administração regista o pagamento em comissoes/pagamento.
Motorista só trabalha com conta aprovada, carta, seguro e licença do veículo válidos (e inspeção, se indicada). Alterar carta, BI, matrícula ou validades volta a pôr a conta "pendente".
Uma corrida ativa por cliente e por motorista (garantido pela base de dados).
Pedidos sem motorista são cancelados após TEMPO_MAX_PEDIDO_MIN; motoristas sem sinal há OFFLINE_APOS_MIN passam a offline.
O que falta antes de abrir a utilizadores reais
SMS real em src/sms.js (agregador com cobertura em Moçambique). Sem isto ninguém consegue entrar.
HTTPS (nginx ou Caddy à frente) e TRUST_PROXY=true; segredo JWT_SEGREDO longo e aleatório; cópias de segurança da base de dados.
M-Pesa / e-Mola (pagamentos e cobrança de comissões) — exige contrato com as operadoras.
Aplicações móveis e painel web que usem esta API (as telas estão em fluxo-telas.md).
Termos de uso e política de privacidade (a API guarda telefone, localização e documentos dos motoristas) e a conversa com o município sobre licenciamento (ver plano-lancamento.md).
Rever limites de pedidos por IP (src/app.js): redes móveis partilham o mesmo IP entre muitas pessoas.
Nota sobre pagamentos de comissões: o saldo do motorista (saldo_comissoes) é a referência e baixa sempre pelo valor exato pago. As marcas "paga" em cada comissão são só um registo auxiliar: comissoes/pagamento marca, por ordem, as comissões que cabem inteiras no valor pago; um pagamento parcial reduz o saldo mas pode não marcar nenhuma.
