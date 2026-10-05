/**
 * ============================================================
 *  FINANÇAS DA FAMÍLIA — Backend (Google Apps Script)
 *  Etapa 1: login Google, estrutura da planilha, API segura,
 *           permissões, auditoria, lançamentos e dashboard.
 *  Etapa 2: contas a pagar/receber com pagamentos parciais,
 *           cartões e faturas, transferências, saldos por conta,
 *           lançamentos recorrentes e parcelados, categorias.
 *  Etapa 3: orçamento por categoria, metas, dívidas com juros e
 *           amortização, relatórios com comparação e exportação.
 *  Etapa 4: central de notificações, duplicados, histórico na tela,
 *           cópias de segurança, automação diária e revisão de acesso.
 * ============================================================
 *
 *  Arquitetura (camadas separadas para facilitar migração futura):
 *    1. Web app ............ doGet / include
 *    2. API ................ api(acao, dados)  ← único ponto de entrada
 *    3. Autenticação ....... emailAtual_, ctx_, exigir_, podeAlterar_
 *    4. Regras de negócio .. acao*_ (salvar, dashboard, membros…)
 *    5. Armazenamento ...... tabela_, inserirVarios_, atualizar_
 *                            (trocar esta camada = trocar o banco)
 */

const VERSAO_APP = '4.0.0 · Etapa 4';
const VERSAO_ESTRUTURA = '3';
const FUSO = 'America/Sao_Paulo';

/** Colunas de controle presentes em quase todas as abas. */
const COLS_CONTROLE = ['demo', 'excluido', 'criado_por', 'criado_em', 'atualizado_por', 'atualizado_em', 'versao'];

/** As 17 abas criadas automaticamente, na ordem da especificação. */
const ENTIDADES = {
  usuarios:        { aba: 'Usuários',          cols: ['id', 'familia_id', 'email', 'nome', 'perfil', 'status', 'editar_todos', 'convidado_por', 'ultimo_acesso'] },
  familias:        { aba: 'Famílias',          cols: ['id', 'nome', 'moeda', 'meta_economia_mensal', 'base_comprometimento'] },
  receitas:        { aba: 'Receitas',          cols: ['id', 'familia_id', 'descricao', 'valor', 'categoria', 'responsavel', 'conta_id', 'data', 'status', 'recorrencia', 'observacoes', 'anexo_url', 'valor_recebido', 'pagamentos', 'serie_id', 'auto'] },
  despesas:        { aba: 'Despesas',          cols: ['id', 'familia_id', 'descricao', 'valor', 'categoria', 'subcategoria', 'responsavel', 'forma_pagamento', 'conta_id', 'data_compra', 'data_vencimento', 'data_pagamento', 'status', 'recorrencia', 'parcela', 'observacoes', 'anexo_url', 'valor_pago', 'pagamentos', 'serie_id', 'auto', 'grupo_id', 'cartao_id', 'compra_id', 'fatura'] },
  contas_pagar:    { aba: 'Contas a pagar',    cols: ['id', 'familia_id', 'descricao', 'valor', 'valor_pago', 'vencimento', 'responsavel', 'categoria', 'status', 'despesa_id', 'observacoes'] },
  contas_receber:  { aba: 'Contas a receber',  cols: ['id', 'familia_id', 'descricao', 'valor', 'valor_recebido', 'vencimento', 'responsavel', 'categoria', 'status', 'receita_id', 'observacoes'] },
  cartoes:         { aba: 'Cartões',           cols: ['id', 'familia_id', 'nome', 'instituicao', 'titular', 'limite', 'dia_fechamento', 'dia_vencimento', 'conta_pagamento_id', 'cor', 'status'] },
  compras_cartao:  { aba: 'Compras no cartão', cols: ['id', 'familia_id', 'cartao_id', 'descricao', 'valor_total', 'parcelas', 'valor_parcela', 'categoria', 'subcategoria', 'data_compra', 'responsavel', 'observacoes', 'primeira_fatura'] },
  faturas:         { aba: 'Faturas',           cols: ['id', 'familia_id', 'cartao_id', 'competencia', 'fechamento', 'vencimento', 'valor_total', 'valor_pago', 'status', 'pagamentos'] },
  contas_bancarias:{ aba: 'Contas bancárias',  cols: ['id', 'familia_id', 'nome', 'instituicao', 'titular', 'tipo', 'saldo_inicial', 'data_saldo_inicial', 'status'] },
  transferencias:  { aba: 'Transferências',    cols: ['id', 'familia_id', 'pessoa_origem', 'pessoa_destino', 'conta_origem_id', 'conta_destino_id', 'valor', 'data', 'descricao'] },
  orcamentos:      { aba: 'Orçamentos',        cols: ['id', 'familia_id', 'competencia', 'categoria', 'subcategoria', 'valor_planejado'] },
  metas:           { aba: 'Metas financeiras', cols: ['id', 'familia_id', 'nome', 'tipo', 'valor_desejado', 'valor_acumulado', 'data_inicio', 'prazo', 'responsavel', 'prioridade', 'descricao', 'status', 'valor_inicial', 'movimentos', 'conta_id'] },
  dividas:         { aba: 'Dívidas',           cols: ['id', 'familia_id', 'nome', 'credor', 'valor_original', 'saldo_devedor', 'taxa_juros', 'qtd_parcelas', 'valor_parcela', 'parcelas_pagas', 'data_contratacao', 'dia_vencimento', 'status', 'tipo', 'conta_id', 'saldo_inicial', 'parcelas_pagas_inicial', 'valor_parcela_inicial', 'parcelas_restantes', 'pagamentos'] },
  categorias:      { aba: 'Categorias',        cols: ['id', 'familia_id', 'tipo', 'grupo', 'nome', 'cor', 'icone', 'padrao', 'ativo'] },
  historico:       { aba: 'Histórico de alterações', cols: ['id', 'familia_id', 'data_hora', 'usuario', 'acao', 'entidade', 'registro_id', 'detalhes'], semControle: true },
  configuracoes:   { aba: 'Configurações',     cols: ['id', 'familia_id', 'chave', 'valor'] }
};

const PERFIS = ['admin', 'membro', 'visualizador'];
const EDITAVEIS = ['receitas', 'despesas', 'contas_bancarias', 'categorias', 'cartoes', 'transferencias', 'orcamentos', 'metas', 'dividas'];
const SO_ADMIN = ['contas_bancarias', 'categorias', 'cartoes'];
const LEGIVEIS = ['receitas', 'despesas', 'contas_bancarias', 'categorias', 'transferencias', 'dividas', 'metas', 'orcamentos', 'cartoes', 'compras_cartao'];
const MUTACOES = ['criarFamilia', 'aceitarConvite', 'convidarMembro', 'alterarMembro', 'salvarFamilia', 'salvar', 'excluir', 'restaurar', 'gerarDemo', 'removerDemo',
  'pagar', 'removerPagamento', 'salvarCompra', 'editarCompra', 'excluirCompra', 'restaurarCompra', 'adiarVencimento',
  'copiarOrcamento', 'movimentarMeta', 'removerMovMeta', 'pagarDivida', 'removerPagDivida',
  'backup', 'configSistema', 'rodarAutomacao', 'corrigirAcesso', 'proteger'];
/** Numa parcela de cartão estes campos são definidos pela compra/fatura e não podem ser editados avulsos. */
const TRAVADOS_CARTAO = ['valor', 'data_compra', 'data_vencimento', 'data_pagamento', 'status', 'conta_id', 'forma_pagamento', 'recorrencia', 'parcela'];
const RECORRENCIAS = ['nenhuma', 'semanal', 'quinzenal', 'mensal', 'trimestral', 'anual'];
const FORMAS_PAGAMENTO = ['', 'dinheiro', 'pix', 'debito', 'credito', 'boleto', 'transferencia', 'outro'];
const TIPOS_CONTA = ['corrente', 'poupanca', 'carteira', 'digital', 'investimentos', 'outra'];

/** Cache válido apenas durante uma execução (não compartilhado entre usuários). */
const _cache = {};

// ============================================================
// 1. WEB APP
// ============================================================

function doGet() {
  return HtmlService.createTemplateFromFile('Index').evaluate()
    .setTitle('Finanças da Família')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1, viewport-fit=cover');
}

function include(nome) {
  return HtmlService.createHtmlOutputFromFile(nome).getContent();
}

/**
 * Execute UMA VEZ pelo editor do Apps Script (menu Executar ▸ instalar).
 * Vincula a planilha ao sistema e cria as 17 abas.
 */
function instalar() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  if (!ss) throw new Error('Abra o Apps Script pela planilha (Extensões ▸ Apps Script) e execute novamente.');
  PropertiesService.getScriptProperties().setProperty('PLANILHA_ID', ss.getId());
  _cache.ss = ss;
  garantirEstrutura_(true);
  Logger.log('Pronto! Estrutura criada na planilha: ' + ss.getUrl());
  Logger.log('Agora implante como App da Web (veja o LEIAME).');
}

// ============================================================
// 2. API — único ponto de entrada chamado pelo navegador
// ============================================================

const ACOES = {
  iniciar: acaoIniciar_,
  versaoDados: acaoVersaoDados_,
  criarFamilia: acaoCriarFamilia_,
  aceitarConvite: acaoAceitarConvite_,
  convidarMembro: acaoConvidar_,
  alterarMembro: acaoAlterarMembro_,
  salvarFamilia: acaoSalvarFamilia_,
  listar: acaoListar_,
  salvar: acaoSalvar_,
  excluir: function (d) { return acaoExcluir_(d, false); },
  restaurar: function (d) { return acaoExcluir_(d, true); },
  dashboard: acaoDashboard_,
  gerarDemo: acaoGerarDemo_,
  removerDemo: acaoRemoverDemo_,
  agenda: acaoAgenda_,
  contas: acaoContas_,
  cartoes: acaoCartoes_,
  pagar: acaoPagar_,
  removerPagamento: acaoRemoverPagamento_,
  adiarVencimento: acaoAdiar_,
  salvarCompra: acaoSalvarCompra_,
  editarCompra: acaoEditarCompra_,
  excluirCompra: function (d) { return acaoExcluirCompra_(d, false); },
  restaurarCompra: function (d) { return acaoExcluirCompra_(d, true); },
  orcamento: acaoOrcamento_,
  copiarOrcamento: acaoCopiarOrcamento_,
  planejamento: acaoPlanejamento_,
  movimentarMeta: acaoMovimentarMeta_,
  removerMovMeta: acaoRemoverMovMeta_,
  pagarDivida: acaoPagarDivida_,
  removerPagDivida: acaoRemoverPagDivida_,
  relatorio: acaoRelatorio_,
  exportarRelatorio: acaoExportarRelatorio_,
  notificacoes: acaoNotificacoes_,
  marcarNotificacoes: acaoMarcarNotificacoes_,
  prefsNotificacoes: acaoPrefsNotificacoes_,
  historico: acaoHistorico_,
  backup: acaoBackup_,
  configSistema: acaoConfigSistema_,
  rodarAutomacao: acaoRodarAutomacao_,
  seguranca: acaoSeguranca_,
  corrigirAcesso: acaoCorrigirAcesso_,
  proteger: acaoProteger_
};

/**
 * Toda chamada do navegador passa por aqui.
 * - Operações de escrita exigem um opId: se a mesma operação chegar de novo
 *   (nova tentativa após falha de conexão), a resposta anterior é devolvida
 *   sem gravar outra vez.
 * - Retorna sempre JSON em texto: {ok, data, versaoDados} ou {ok:false, erro}.
 */
function api(acao, dados) {
  dados = dados || {};
  try {
    const fn = ACOES[acao];
    if (!fn) throw erro_('ACAO_INVALIDA', 'Ação desconhecida.');
    let chaveOp = null;
    if (MUTACOES.indexOf(acao) >= 0) {
      if (!dados.opId || !/^[\w-]{8,64}$/.test(String(dados.opId))) throw erro_('OP_ID', 'Operação sem identificador.');
      chaveOp = 'op:' + emailAtual_() + ':' + dados.opId;
      const anterior = CacheService.getScriptCache().get(chaveOp);
      if (anterior) return anterior;
    }
    const data = fn(dados);
    const resposta = JSON.stringify({ ok: true, data: data, versaoDados: versaoDados_() });
    if (chaveOp && resposta.length < 90000) CacheService.getScriptCache().put(chaveOp, resposta, 21600);
    return resposta;
  } catch (e) {
    const conhecido = !!e.codigo;
    // Log sem dados financeiros: apenas ação e mensagem técnica.
    if (!conhecido) console.error('Erro em ' + acao + ': ' + (e && e.message));
    return JSON.stringify({
      ok: false,
      erro: {
        codigo: e.codigo || 'ERRO_INTERNO',
        mensagem: conhecido ? e.message : 'Ocorreu um erro inesperado. Tente novamente; se continuar, avise o administrador.'
      }
    });
  }
}

// ============================================================
// 3. AUTENTICAÇÃO E PERMISSÕES
// ============================================================

function emailAtual_() {
  const e = String(Session.getActiveUser().getEmail() || '').trim().toLowerCase();
  if (!e) throw erro_('SEM_IDENTIDADE', 'Não foi possível identificar sua conta Google. Entre com sua conta Google e abra o link novamente.');
  return e;
}

function familia_() {
  return tabela_('familias').rows.find(function (r) { return r.excluido !== '1'; }) || null;
}

function usuarioPorEmail_(email, familiaId) {
  return tabela_('usuarios').rows.find(function (r) {
    return r.email.toLowerCase() === email && r.familia_id === familiaId && r.excluido !== '1';
  }) || null;
}

/** Contexto da requisição: quem é, de qual família, com qual perfil. */
function ctx_() {
  const email = emailAtual_();
  const fam = familia_();
  if (!fam) throw erro_('SEM_FAMILIA', 'A família ainda não foi criada.');
  const u = usuarioPorEmail_(email, fam.id);
  if (!u || u.status !== 'ativo') throw erro_('SEM_PERMISSAO', 'Seu acesso a esta família não está ativo.');
  return { email: email, usuario: u, familia: fam, perfil: u.perfil };
}

function exigir_(c, permissao) {
  const regras = {
    ler: true,
    criar: c.perfil === 'admin' || c.perfil === 'membro',
    administrar: c.perfil === 'admin'
  };
  if (!regras[permissao]) {
    throw erro_('SEM_PERMISSAO', permissao === 'administrar'
      ? 'Somente o administrador pode fazer isso.'
      : 'Seu perfil permite apenas consultar os dados.');
  }
}

/** Membro altera os próprios lançamentos; os demais só se o admin permitir. */
function podeAlterar_(c, reg) {
  if (c.perfil === 'admin') return true;
  if (c.perfil !== 'membro') return false;
  return reg.criado_por === c.email || c.usuario.editar_todos === '1';
}

function ehDono_(email) {
  try {
    const dono = ss_().getOwner();
    return !dono || String(dono.getEmail()).toLowerCase() === email;
  } catch (e) {
    return true;
  }
}

// ============================================================
// 4. AÇÕES — INÍCIO, FAMÍLIA E MEMBROS
// ============================================================

function acaoIniciar_() {
  const email = emailAtual_();
  try {
    ss_();
  } catch (e) {
    return { estado: 'negado', email: email, motivo: e.message };
  }
  try { garantirEstrutura_(false); } catch (e) { /* visualizador não altera estrutura */ }

  const fam = familia_();
  if (!fam) return { estado: 'configurar', email: email, podeCriar: ehDono_(email) };

  const u = usuarioPorEmail_(email, fam.id);
  if (!u || u.status === 'revogado') {
    return { estado: 'negado', email: email, motivo: 'Este e-mail não faz parte da família "' + fam.nome + '". Peça ao administrador um convite.' };
  }
  if (u.status === 'convidado') {
    return {
      estado: 'convite', email: email,
      familia: { nome: fam.nome },
      convite: { perfil: u.perfil, nome: u.nome, convidado_por: nomeDe_(u.convidado_por, fam.id) }
    };
  }
  const c = { email: email, usuario: u, familia: fam, perfil: u.perfil };
  registrarAcesso_(c);
  if (c.perfil !== 'visualizador') rodarRecorrenciasDoDia_(c);
  return Object.assign({ estado: 'pronto' }, pacote_(c));
}

function acaoVersaoDados_() {
  emailAtual_();
  return versaoDados_();
}

function acaoCriarFamilia_(d) {
  const email = emailAtual_();
  const nomeFam = texto_(d.nomeFamilia, 'Nome da família', 80, true);
  const nomeUsu = texto_(d.nomeUsuario, 'Seu nome', 60, true);
  const conta = d.conta && d.conta.nome ? validarConta_(d.conta) : null;
  return comLock_(function () {
    garantirEstrutura_(true);
    if (familia_()) throw erro_('JA_EXISTE', 'A família já foi criada. Recarregue a página.');
    if (!ehDono_(email)) throw erro_('SEM_PERMISSAO', 'Somente o dono da planilha pode criar a família.');

    const fam = marcarCriacao_({ id: uuid_(), nome: nomeFam, moeda: 'BRL', meta_economia_mensal: '', base_comprometimento: 'assumidas' }, email);
    inserirVarios_(tabela_('familias'), [fam]);
    const u = marcarCriacao_({
      id: uuid_(), familia_id: fam.id, email: email, nome: nomeUsu, perfil: 'admin',
      status: 'ativo', editar_todos: '1', convidado_por: '', ultimo_acesso: agora_()
    }, email);
    inserirVarios_(tabela_('usuarios'), [u]);
    const c = { email: email, usuario: u, familia: fam, perfil: 'admin' };

    inserirVarios_(tabela_('categorias'), categoriasPadrao_(fam.id, email));
    if (conta) {
      inserirVarios_(tabela_('contas_bancarias'), [marcarCriacao_(Object.assign(conta, { id: uuid_(), familia_id: fam.id }), email)]);
    }
    auditar_(c, 'criar', 'familias', fam.id, { nome: nomeFam });
    if (d.demo) gerarDemo_(c);
    marcarAlteracao_();
    return Object.assign({ estado: 'pronto' }, pacote_(c));
  });
}

function acaoAceitarConvite_(d) {
  const email = emailAtual_();
  return comLock_(function () {
    const fam = familia_();
    if (!fam) throw erro_('SEM_FAMILIA', 'A família não foi encontrada.');
    const t = tabela_('usuarios');
    const u = usuarioPorEmail_(email, fam.id);
    if (!u || u.status !== 'convidado') throw erro_('SEM_CONVITE', 'Não há convite pendente para este e-mail.');
    const nome = texto_(d.nome, 'Seu nome', 60, false) || u.nome;
    Object.assign(u, { nome: nome, status: 'ativo', ultimo_acesso: agora_(), atualizado_por: email, atualizado_em: agora_(), versao: proximaVersao_(u) });
    atualizar_(t, u);
    const c = { email: email, usuario: u, familia: fam, perfil: u.perfil };
    auditar_(c, 'aceitar_convite', 'usuarios', u.id, { perfil: u.perfil });
    marcarAlteracao_();
    return Object.assign({ estado: 'pronto' }, pacote_(c));
  });
}

function acaoConvidar_(d) {
  const c = ctx_();
  exigir_(c, 'administrar');
  const email = String(d.email || '').trim().toLowerCase();
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) throw erro_('VALIDACAO', 'Informe um e-mail válido.');
  const perfil = opcao_(d.perfil, PERFIS, 'membro', 'Perfil');
  const nome = texto_(d.nome, 'Nome', 60, false) || email.split('@')[0];
  return comLock_(function () {
    const t = tabela_('usuarios');
    let u = usuarioPorEmail_(email, c.familia.id);
    if (u && u.status !== 'revogado') throw erro_('JA_EXISTE', 'Este e-mail já faz parte da família ou já tem um convite pendente.');
    const aviso = compartilhar_(email, perfil);
    if (u) {
      Object.assign(u, { status: 'convidado', perfil: perfil, nome: nome, convidado_por: c.email, atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(u) });
      atualizar_(t, u);
    } else {
      u = marcarCriacao_({
        id: uuid_(), familia_id: c.familia.id, email: email, nome: nome, perfil: perfil,
        status: 'convidado', editar_todos: '1', convidado_por: c.email, ultimo_acesso: ''
      }, c.email);
      inserirVarios_(t, [u]);
    }
    auditar_(c, 'convidar', 'usuarios', u.id, { email: email, perfil: perfil });
    marcarAlteracao_();
    return { membros: membros_(c.familia.id), urlApp: urlApp_(), aviso: aviso };
  });
}

function acaoAlterarMembro_(d) {
  const c = ctx_();
  exigir_(c, 'administrar');
  return comLock_(function () {
    const t = tabela_('usuarios');
    const u = t.rows.find(function (r) { return r.id === d.id && r.familia_id === c.familia.id && r.excluido !== '1'; });
    if (!u) throw erro_('NAO_ENCONTRADO', 'Membro não encontrado.');
    if (u.email === c.email) throw erro_('VALIDACAO', 'Você não pode alterar o próprio perfil ou revogar o próprio acesso.');

    const mud = {};
    if (d.perfil !== undefined) {
      const p = opcao_(d.perfil, PERFIS, null, 'Perfil');
      if (p !== u.perfil) { mud.perfil = [u.perfil, p]; u.perfil = p; if (u.status !== 'revogado') compartilhar_(u.email, p); }
    }
    if (d.editar_todos !== undefined) {
      const v = d.editar_todos ? '1' : '0';
      if (v !== u.editar_todos) { mud.editar_todos = [u.editar_todos, v]; u.editar_todos = v; }
    }
    if (d.status === 'revogado' && u.status !== 'revogado') {
      mud.status = [u.status, 'revogado'];
      u.status = 'revogado';
      descompartilhar_(u.email);
    }
    if (!Object.keys(mud).length) return { membros: membros_(c.familia.id) };
    Object.assign(u, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(u) });
    atualizar_(t, u);
    auditar_(c, 'alterar_membro', 'usuarios', u.id, mud);
    marcarAlteracao_();
    return { membros: membros_(c.familia.id) };
  });
}

function acaoSalvarFamilia_(d) {
  const c = ctx_();
  exigir_(c, 'administrar');
  const nome = texto_(d.nome, 'Nome da família', 80, true);
  const meta = (d.meta_economia_mensal === '' || d.meta_economia_mensal == null)
    ? '' : paraTexto_(centavos_(d.meta_economia_mensal, 'Meta de economia', { zero: true }));
  const base = opcao_(d.base_comprometimento, ['assumidas', 'pagas'], 'assumidas', 'Base do comprometimento');
  return comLock_(function () {
    const t = tabela_('familias');
    const f = t.rows.find(function (r) { return r.id === c.familia.id; });
    const mud = {};
    [['nome', nome], ['meta_economia_mensal', meta], ['base_comprometimento', base]].forEach(function (p) {
      if (String(f[p[0]] || '') !== p[1]) { mud[p[0]] = [f[p[0]] || '', p[1]]; f[p[0]] = p[1]; }
    });
    if (Object.keys(mud).length) {
      Object.assign(f, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(f) });
      atualizar_(t, f);
      auditar_(c, 'editar', 'familias', f.id, mud);
      marcarAlteracao_();
    }
    c.familia = f;
    return pacote_(c);
  });
}

function compartilhar_(email, perfil) {
  const ss = ss_();
  try {
    if (perfil === 'visualizador') {
      try { ss.removeEditor(email); } catch (e) { /* não era editor */ }
      ss.addViewer(email);
    } else {
      ss.addEditor(email);
    }
    return '';
  } catch (e) {
    return 'Não consegui compartilhar a planilha automaticamente. Compartilhe-a manualmente com ' + email +
      ' como ' + (perfil === 'visualizador' ? 'leitor' : 'editor') + '.';
  }
}

function descompartilhar_(email) {
  const ss = ss_();
  try { ss.removeEditor(email); } catch (e) { /* ignora */ }
  try { ss.removeViewer(email); } catch (e) { /* ignora */ }
}

function registrarAcesso_(c) {
  if (c.perfil === 'visualizador') return;
  const ultimo = c.usuario.ultimo_acesso || '';
  const umaHoraAtras = Utilities.formatDate(new Date(Date.now() - 3600000), FUSO, "yyyy-MM-dd'T'HH:mm:ss");
  if (ultimo > umaHoraAtras) return;
  try {
    const t = tabela_('usuarios');
    const col = t.header.indexOf('ultimo_acesso');
    if (col >= 0) t.sh.getRange(c.usuario._linha, col + 1).setNumberFormat('@').setValue(agora_());
  } catch (e) { /* não crítico */ }
}

// ============================================================
// 5. AÇÕES — LANÇAMENTOS (listar, salvar, excluir, restaurar)
// ============================================================

function acaoListar_(d) {
  const c = ctx_();
  const ent = d.entidade;
  if (LEGIVEIS.indexOf(ent) < 0) throw erro_('ACAO_INVALIDA', 'Consulta não permitida.');
  let rows = ativos_(ent, c.familia.id);
  const campo = ent === 'receitas' ? 'data' : ent === 'despesas' ? 'data_compra' : ent === 'transferencias' ? 'data' : null;
  if (campo && d.inicio && d.fim) {
    const ini = dataISO_(d.inicio, 'Início', true), fim = dataISO_(d.fim, 'Fim', true);
    rows = rows.filter(function (r) {
      const x = r[campo] || r.data_vencimento || '';
      return x >= ini && x <= fim;
    });
  }
  if (campo) rows.sort(function (a, b) { return (b[campo] || '').localeCompare(a[campo] || ''); });
  return rows.map(publico_);
}

function acaoSalvar_(d) {
  const c = ctx_();
  const ent = d.entidade;
  if (EDITAVEIS.indexOf(ent) < 0) throw erro_('ACAO_INVALIDA', 'Este tipo de registro não pode ser editado por aqui.');
  exigir_(c, SO_ADMIN.indexOf(ent) >= 0 ? 'administrar' : 'criar');
  const bruto = d.registro || {};
  const id = String(bruto.id || '');
  if (!uuidValido_(id)) throw erro_('VALIDACAO', 'Identificador inválido.');
  const limpo = validar_(ent, bruto, c);
  const nParc = (ent === 'despesas' && d.modo === 'criar') ? inteiro_(d.parcelas, 1, 1, 72, 'Parcelas') : 1;
  if (nParc > 1 && limpo.recorrencia !== 'nenhuma') throw erro_('VALIDACAO', 'Uma despesa parcelada não pode ser também recorrente.');
  const lancamento = ent === 'receitas' || ent === 'despesas';
  const campoData = ent === 'receitas' ? 'data' : 'data_compra';

  return comLock_(function () {
    const t = tabela_(ent);
    const existente = t.rows.find(function (r) { return r.id === id; });
    if (existente && existente.familia_id !== c.familia.id) throw erro_('SEM_PERMISSAO', 'Registro não pertence à sua família.');

    if (d.modo === 'criar') {
      // Mesmo id já gravado = nova tentativa da mesma operação → não duplica.
      if (existente) return { registro: publico_(existente), repetido: true };
      if (lancamento && limpo.recorrencia !== 'nenhuma') limpo.serie_id = id;
      if (ent === 'orcamentos') orcamentoUnico_(c, limpo, id);
      calcular_(ent, limpo);
      const novos = nParc > 1 ? parcelar_(limpo, id, nParc, c)
        : [marcarCriacao_(Object.assign(limpo, { id: id, familia_id: c.familia.id }), c.email)];
      inserirVarios_(t, novos);
      auditar_(c, 'criar', ent, id, Object.assign(resumo_(novos[0]), nParc > 1 ? { parcelas: nParc } : {}));
      const gerados = limpo.serie_id ? gerarRecorrencias_(c, { ent: ent, serie: id }) : 0;
      marcarAlteracao_();
      return { registro: publico_(novos[0]), parcelas: nParc, gerados: gerados };
    }

    if (d.modo !== 'editar') throw erro_('ACAO_INVALIDA', 'Modo inválido.');
    if (!existente || existente.excluido === '1') throw erro_('NAO_ENCONTRADO', 'Registro não encontrado. Ele pode ter sido excluído por outra pessoa.');
    if (!podeAlterar_(c, existente)) throw erro_('SEM_PERMISSAO', 'Você não tem permissão para editar lançamentos de outras pessoas.');
    if (String(bruto.versao) !== String(existente.versao)) {
      throw erro_('CONFLITO', 'Outra pessoa alterou este registro enquanto você editava. Os dados foram recarregados — confira e salve de novo.');
    }
    if (ent === 'despesas' && existente.cartao_id) TRAVADOS_CARTAO.forEach(function (k) { limpo[k] = existente[k]; });
    const novo = Object.assign({}, existente, limpo);
    if (lancamento && pags_(existente).length && novo.status !== 'cancelada') aplicarStatus_(novo, ent);
    if (ent === 'orcamentos') orcamentoUnico_(c, novo, id);
    calcular_(ent, novo);

    const extra = {};
    let gerarSerie = '';
    if (lancamento && !existente.cartao_id) {
      const antes = existente.recorrencia || 'nenhuma', depois = novo.recorrencia || 'nenhuma';
      if (antes !== depois) {
        if (existente.serie_id) {
          // Encerra a série antiga a partir deste lançamento e remove as próximas ocorrências automáticas em aberto.
          setConfig_(c, 'serie_fim:' + existente.serie_id, depois === 'nenhuma' ? novo[campoData] : addDias_(novo[campoData], -1));
          extra.encerradas = encerrarFuturas_(c, ent, existente.serie_id, novo[campoData], id);
        }
        if (depois !== 'nenhuma') {
          novo.serie_id = existente.serie_id === id ? uuid_() : id;
          gerarSerie = novo.serie_id;
        }
      } else if (d.aplicarProximas && existente.serie_id) {
        extra.atualizadas = aplicarProximas_(c, ent, novo, campoData);
      }
    }
    const mud = {};
    colunas_(ent).forEach(function (k) {
      if (COLS_CONTROLE.indexOf(k) >= 0 || k === 'id' || k === 'familia_id') return;
      if (String(existente[k] || '') !== String(novo[k] || '')) mud[k] = [existente[k] || '', novo[k] || ''];
    });
    if (!Object.keys(mud).length && !extra.atualizadas && !extra.encerradas) return { registro: publico_(existente), semMudancas: true };
    if (Object.keys(mud).length) {
      Object.assign(existente, novo, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(existente) });
      atualizar_(t, existente);
      auditar_(c, 'editar', ent, id, mud);
    }
    if (gerarSerie) extra.gerados = gerarRecorrencias_(c, { ent: ent, serie: gerarSerie });
    marcarAlteracao_();
    return Object.assign({ registro: publico_(existente) }, extra);
  });
}

/** Exclusão lógica: o registro fica na planilha marcado, preservando a auditoria. */
function acaoExcluir_(d, restaurar) {
  const c = ctx_();
  const ent = d.entidade;
  if (EDITAVEIS.indexOf(ent) < 0) throw erro_('ACAO_INVALIDA', 'Exclusão não permitida.');
  exigir_(c, SO_ADMIN.indexOf(ent) >= 0 ? 'administrar' : 'criar');
  return comLock_(function () {
    const t = tabela_(ent);
    const r = t.rows.find(function (x) { return x.id === d.id && x.familia_id === c.familia.id; });
    if (!r) throw erro_('NAO_ENCONTRADO', 'Registro não encontrado.');
    if (!podeAlterar_(c, r)) throw erro_('SEM_PERMISSAO', 'Você não tem permissão para excluir este lançamento.');
    if (ent === 'despesas' && r.cartao_id) throw erro_('VALIDACAO', 'Esta é uma parcela de cartão. Exclua a compra inteira na tela Cartões.');
    if (ent === 'cartoes' && !restaurar && ativos_('despesas', c.familia.id).some(function (x) { return x.cartao_id === r.id; })) {
      throw erro_('VALIDACAO', 'Este cartão tem compras registradas. Em vez de excluir, edite o cartão e marque-o como inativo.');
    }
    const alvo = restaurar ? '' : '1';
    if ((r.excluido || '') === alvo) return { registro: publico_(r) };
    Object.assign(r, { excluido: alvo, atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(r) });
    atualizar_(t, r);
    auditar_(c, restaurar ? 'restaurar' : 'excluir', ent, r.id, resumo_(r));
    marcarAlteracao_();
    return { registro: publico_(r) };
  });
}

/** Divide uma despesa em N parcelas mensais (boleto, carnê…). O valor informado é o total. */
function parcelar_(limpo, id, n, c) {
  const total = cent_(limpo.valor);
  if (total < n) throw erro_('VALIDACAO', 'Valor muito baixo para o número de parcelas.');
  const base = Math.floor(total / n), resto = total - base * n;
  const venc0 = limpo.data_vencimento || limpo.data_compra;
  const out = [];
  for (let i = 0; i < n; i++) {
    const o = Object.assign({}, limpo, {
      id: i === 0 ? id : uuid_(), familia_id: c.familia.id, grupo_id: id,
      descricao: (limpo.descricao + ' (' + (i + 1) + '/' + n + ')').slice(0, 130),
      parcela: (i + 1) + '/' + n,
      valor: paraTexto_(base + (i === 0 ? resto : 0)),
      data_compra: addMesesDia_(limpo.data_compra, i),
      data_vencimento: addMesesDia_(venc0, i)
    });
    if (i > 0) { o.status = 'pendente'; o.data_pagamento = ''; }
    out.push(marcarCriacao_(o, c.email));
  }
  return out;
}

function acaoAdiar_(d) {
  const c = ctx_();
  exigir_(c, 'criar');
  const nova = dataISO_(d.data, 'Novo vencimento', true);
  return comLock_(function () {
    const t = tabela_('despesas');
    const r = t.rows.find(function (x) { return x.id === d.id && x.familia_id === c.familia.id && x.excluido !== '1'; });
    if (!r) throw erro_('NAO_ENCONTRADO', 'Conta não encontrada.');
    if (!podeAlterar_(c, r)) throw erro_('SEM_PERMISSAO', 'Você não tem permissão para alterar esta conta.');
    if (r.cartao_id) throw erro_('VALIDACAO', 'Parcelas de cartão seguem a data da fatura.');
    if (restante_(r, 'despesas') <= 0) throw erro_('VALIDACAO', 'Esta conta já está paga.');
    const antes = r.data_vencimento || r.data_compra;
    Object.assign(r, { data_vencimento: nova, atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(r) });
    atualizar_(t, r);
    auditar_(c, 'adiar', 'despesas', r.id, { data_vencimento: [antes, nova] });
    marcarAlteracao_();
    return { registro: publico_(r) };
  });
}

// ------------------------------------------------------------
// Pagamentos (totais ou parciais) de despesas, receitas e faturas
// ------------------------------------------------------------

function pags_(r) {
  if (!r || !r.pagamentos) return [];
  try { const a = JSON.parse(r.pagamentos); return Array.isArray(a) ? a : []; } catch (e) { return []; }
}
function somaPags_(r) { return pags_(r).reduce(function (s, p) { return s + cent_(p.valor); }, 0); }

/** Quanto do lançamento já foi pago/recebido (centavos). */
function quitado_(r, ent) {
  if (r.status === 'cancelada') return 0;
  const v = cent_(r.valor);
  if (ent === 'despesas' && r.cartao_id) return r.status === 'paga' ? v : 0;
  if (pags_(r).length) return Math.min(v, somaPags_(r));
  return (r.status === 'paga' || r.status === 'recebida') ? v : 0;
}
function restante_(r, ent) { return r.status === 'cancelada' ? 0 : Math.max(0, cent_(r.valor) - quitado_(r, ent)); }

/** Com pagamentos registrados, a situação é calculada a partir deles. */
function aplicarStatus_(r, ent) {
  const p = pags_(r);
  if (r.status === 'cancelada') return;
  const soma = somaPags_(r), v = cent_(r.valor);
  if (ent === 'despesas') {
    r.valor_pago = p.length ? paraTexto_(soma) : '';
    if (!p.length) { r.status = 'pendente'; r.data_pagamento = ''; return; }
    if (soma >= v) { r.status = 'paga'; r.data_pagamento = p.map(function (x) { return x.data; }).sort().pop(); }
    else { r.status = 'pendente'; r.data_pagamento = ''; }
  } else {
    r.valor_recebido = p.length ? paraTexto_(soma) : '';
    r.status = (p.length && soma >= v) ? 'recebida' : 'prevista';
  }
}

function acaoPagar_(d) {
  const c = ctx_();
  exigir_(c, 'criar');
  const ent = d.entidade;
  if (['despesas', 'receitas', 'faturas'].indexOf(ent) < 0) throw erro_('ACAO_INVALIDA', 'Pagamento inválido.');
  const valor = centavos_(d.valor, 'Valor');
  const data = dataISO_(d.data, 'Data do pagamento', true);
  const conta = contaValida_(d.conta_id, c);
  const fid = c.familia.id;
  return comLock_(function () {
    const pag = { id: uuid_(), data: data, valor: paraTexto_(valor), conta_id: conta, por: c.email, em: agora_() };
    if (ent === 'faturas') {
      const cart = cartaoDe_(d.cartao_id, c);
      const comp = competenciaValida_(d.competencia);
      const fat = sincronizarFatura_(c, cart, comp);
      if (!fat) throw erro_('VALIDACAO', 'Esta fatura não tem compras.');
      const rest = cent_(fat.valor_total) - somaPags_(fat);
      if (rest <= 0) throw erro_('VALIDACAO', 'Esta fatura já está paga.');
      if (valor > rest) throw erro_('VALIDACAO', 'O valor é maior que o restante da fatura (' + brl_(rest) + ').');
      if (!pag.conta_id) pag.conta_id = cart.conta_pagamento_id || '';
      const lista = pags_(fat); lista.push(pag);
      Object.assign(fat, { pagamentos: JSON.stringify(lista), atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(fat) });
      atualizar_(tabela_('faturas'), fat);
      sincronizarFatura_(c, cart, comp);
      auditar_(c, 'pagar', 'faturas', fat.id, { cartao: cart.nome, competencia: comp, valor: pag.valor, data: data });
      marcarAlteracao_();
      return { pagamento: pag, restante: rest - valor };
    }
    const t = tabela_(ent);
    const r = t.rows.find(function (x) { return x.id === d.id && x.familia_id === fid && x.excluido !== '1'; });
    if (!r) throw erro_('NAO_ENCONTRADO', 'Lançamento não encontrado.');
    if (!podeAlterar_(c, r)) throw erro_('SEM_PERMISSAO', 'Você não tem permissão para alterar este lançamento.');
    if (r.cartao_id) throw erro_('VALIDACAO', 'Compras no cartão são quitadas pelo pagamento da fatura.');
    if (r.status === 'cancelada') throw erro_('VALIDACAO', 'Este lançamento está cancelado.');
    const lista = pags_(r);
    if (!lista.length && (r.status === 'paga' || r.status === 'recebida')) throw erro_('VALIDACAO', 'Este lançamento já está quitado.');
    const rest = cent_(r.valor) - somaPags_(r);
    if (rest <= 0) throw erro_('VALIDACAO', 'Este lançamento já está quitado.');
    if (valor > rest) throw erro_('VALIDACAO', 'O valor é maior que o restante (' + brl_(rest) + ').');
    if (!pag.conta_id) pag.conta_id = r.conta_id || '';
    lista.push(pag);
    r.pagamentos = JSON.stringify(lista);
    if (!r.conta_id && pag.conta_id) r.conta_id = pag.conta_id;
    aplicarStatus_(r, ent);
    Object.assign(r, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(r) });
    atualizar_(t, r);
    auditar_(c, ent === 'receitas' ? 'receber' : 'pagar', ent, r.id, { valor: pag.valor, data: data });
    marcarAlteracao_();
    return { registro: publico_(r), restante: rest - valor };
  });
}

function acaoRemoverPagamento_(d) {
  const c = ctx_();
  exigir_(c, 'criar');
  const ent = d.entidade;
  if (['despesas', 'receitas', 'faturas'].indexOf(ent) < 0) throw erro_('ACAO_INVALIDA', 'Operação inválida.');
  return comLock_(function () {
    if (ent === 'faturas') {
      const cart = cartaoDe_(d.cartao_id, c);
      const comp = competenciaValida_(d.competencia);
      const fat = sincronizarFatura_(c, cart, comp);
      const lista = pags_(fat);
      const p = lista.find(function (x) { return x.id === d.pagamento_id; });
      if (!p) throw erro_('NAO_ENCONTRADO', 'Pagamento não encontrado.');
      if (c.perfil !== 'admin' && p.por !== c.email) throw erro_('SEM_PERMISSAO', 'Só quem registrou o pagamento (ou o administrador) pode removê-lo.');
      Object.assign(fat, { pagamentos: JSON.stringify(lista.filter(function (x) { return x !== p; })), atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(fat) });
      atualizar_(tabela_('faturas'), fat);
      sincronizarFatura_(c, cart, comp);
      auditar_(c, 'remover_pagamento', 'faturas', fat.id, { valor: p.valor, data: p.data });
      marcarAlteracao_();
      return { ok: true };
    }
    const t = tabela_(ent);
    const r = t.rows.find(function (x) { return x.id === d.id && x.familia_id === c.familia.id && x.excluido !== '1'; });
    if (!r) throw erro_('NAO_ENCONTRADO', 'Lançamento não encontrado.');
    if (!podeAlterar_(c, r)) throw erro_('SEM_PERMISSAO', 'Você não tem permissão para alterar este lançamento.');
    const lista = pags_(r);
    const p = lista.find(function (x) { return x.id === d.pagamento_id; });
    if (!p) throw erro_('NAO_ENCONTRADO', 'Pagamento não encontrado.');
    const resto = lista.filter(function (x) { return x !== p; });
    r.pagamentos = resto.length ? JSON.stringify(resto) : '';
    aplicarStatus_(r, ent);
    Object.assign(r, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(r) });
    atualizar_(t, r);
    auditar_(c, 'remover_pagamento', ent, r.id, { valor: p.valor, data: p.data });
    marcarAlteracao_();
    return { registro: publico_(r) };
  });
}

// ------------------------------------------------------------
// Cartões de crédito, compras parceladas e faturas
// ------------------------------------------------------------

function cartaoDe_(id, c) {
  const k = tabela_('cartoes').rows.find(function (x) { return x.id === id && x.familia_id === c.familia.id && x.excluido !== '1'; });
  if (!k) throw erro_('NAO_ENCONTRADO', 'Cartão não encontrado.');
  return k;
}

function competenciaValida_(s) {
  s = String(s || '');
  if (!/^\d{4}-(0[1-9]|1[0-2])$/.test(s)) throw erro_('VALIDACAO', 'Fatura inválida.');
  return s;
}

/**
 * Regra das faturas: a fatura é identificada pelo mês do vencimento (AAAA-MM).
 * Compras feitas ANTES do dia de fechamento entram na fatura que fecha naquele mês;
 * no dia do fechamento ou depois, vão para a seguinte.
 */
function competenciaCompra_(k, dataISO) {
  const F = parseInt(k.dia_fechamento, 10) || 1, V = parseInt(k.dia_vencimento, 10) || 10;
  const y = +dataISO.slice(0, 4), m = +dataISO.slice(5, 7), d = +dataISO.slice(8, 10);
  const fechEsteMes = Math.min(F, ultimoDia_(y, m));
  const cm = d < fechEsteMes ? { y: y, m: m } : addMes_(y, m, 1);
  const vm = V > F ? cm : addMes_(cm.y, cm.m, 1);
  return vm.y + '-' + String(vm.m).padStart(2, '0');
}

function datasFatura_(k, comp) {
  const F = parseInt(k.dia_fechamento, 10) || 1, V = parseInt(k.dia_vencimento, 10) || 10;
  const y = +comp.slice(0, 4), m = +comp.slice(5, 7);
  const cm = V > F ? { y: y, m: m } : addMes_(y, m, -1);
  return {
    fechamento: iso_(cm.y, cm.m, Math.min(F, ultimoDia_(cm.y, cm.m))),
    vencimento: iso_(y, m, Math.min(V, ultimoDia_(y, m)))
  };
}

function addComp_(comp, n) {
  const p = addMes_(+comp.slice(0, 4), +comp.slice(5, 7), n);
  return p.y + '-' + String(p.m).padStart(2, '0');
}

function statusFatura_(total, pago, dt, hoje) {
  if (total <= 0 && pago <= 0) return 'vazia';
  if (total - pago <= 0) return 'paga';
  if (hoje < dt.fechamento) return 'aberta';
  if (hoje > dt.vencimento) return 'vencida';
  return 'fechada';
}

/** Recalcula uma fatura a partir das parcelas e dos pagamentos, e acerta a situação das parcelas. */
function sincronizarFatura_(c, k, comp) {
  const fid = c.familia.id;
  const td = tabela_('despesas'), tf = tabela_('faturas');
  const parcelas = td.rows.filter(function (r) {
    return r.familia_id === fid && r.cartao_id === k.id && r.fatura === comp && r.excluido !== '1' && r.status !== 'cancelada';
  });
  let fat = tf.rows.find(function (r) { return r.familia_id === fid && r.cartao_id === k.id && r.competencia === comp && r.excluido !== '1'; }) || null;
  const total = parcelas.reduce(function (s, r) { return s + cent_(r.valor); }, 0);
  const lista = fat ? pags_(fat) : [];
  const pago = lista.reduce(function (s, p) { return s + cent_(p.valor); }, 0);
  const dt = datasFatura_(k, comp);
  const quitada = total > 0 && pago >= total;
  const ult = lista.map(function (p) { return p.data; }).sort().pop() || '';
  parcelas.forEach(function (p) {
    const st = quitada ? 'paga' : 'pendente', dp = quitada ? ult : '';
    if (p.status !== st || (p.data_pagamento || '') !== dp || p.data_vencimento !== dt.vencimento) {
      Object.assign(p, { status: st, data_pagamento: dp, data_vencimento: dt.vencimento, atualizado_em: agora_(), versao: proximaVersao_(p) });
      atualizar_(td, p);
    }
  });
  const campos = {
    valor_total: paraTexto_(total), valor_pago: paraTexto_(pago), status: statusFatura_(total, pago, dt, hoje_()),
    fechamento: dt.fechamento, vencimento: dt.vencimento
  };
  if (!fat) {
    if (!total && !pago) return null;
    fat = marcarCriacao_(Object.assign({ id: uuid_(), familia_id: fid, cartao_id: k.id, competencia: comp, pagamentos: '', demo: k.demo || '' }, campos), c.email);
    inserirVarios_(tf, [fat]);
  } else if (Object.keys(campos).some(function (x) { return String(fat[x] || '') !== campos[x]; })) {
    Object.assign(fat, campos, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(fat) });
    atualizar_(tf, fat);
  }
  return fat;
}

/** Visão calculada de todas as faturas da família. */
function faturasFamilia_(fid) {
  const cartoes = {};
  tabela_('cartoes').rows.forEach(function (k) { if (k.familia_id === fid && k.excluido !== '1') cartoes[k.id] = k; });
  const mapa = {};
  const pegar = function (cid, comp) {
    const k = cid + '|' + comp;
    return mapa[k] || (mapa[k] = { cartao_id: cid, competencia: comp, total: 0, pago: 0, parcelas: [], pagamentos: [] });
  };
  ativos_('despesas', fid).forEach(function (r) {
    if (!r.cartao_id || !r.fatura || r.status === 'cancelada' || !cartoes[r.cartao_id]) return;
    const m = pegar(r.cartao_id, r.fatura);
    m.total += cent_(r.valor);
    m.parcelas.push(r);
  });
  ativos_('faturas', fid).forEach(function (f) {
    if (!cartoes[f.cartao_id]) return;
    const m = pegar(f.cartao_id, f.competencia);
    m.pagamentos = pags_(f);
    m.pago = somaPags_(f);
    m.id = f.id;
  });
  const hoje = hoje_();
  return Object.keys(mapa).map(function (k) {
    const m = mapa[k], cart = cartoes[m.cartao_id], dt = datasFatura_(cart, m.competencia);
    return Object.assign(m, {
      fechamento: dt.fechamento, vencimento: dt.vencimento, restante: m.total - m.pago,
      status: statusFatura_(m.total, m.pago, dt, hoje), cartao_nome: cart.nome, cor: cart.cor, conta_pagamento_id: cart.conta_pagamento_id || ''
    });
  }).filter(function (m) { return m.total > 0 || m.pago > 0; })
    .sort(function (a, b) { return a.vencimento.localeCompare(b.vencimento); });
}

function faturaItem_(f) {
  return {
    _tipo: 'fatura', id: f.cartao_id + '|' + f.competencia, cartao_id: f.cartao_id, competencia: f.competencia,
    descricao: 'Fatura ' + f.cartao_nome + ' · ' + mesAno_(f.competencia), valor: paraTexto_(f.total),
    _restante: Math.max(0, f.restante), _pago: f.pago, _venc: f.vencimento, data_vencimento: f.vencimento,
    fechamento: f.fechamento, status: f.status, categoria: 'Cartão de crédito', responsavel: '', cor: f.cor,
    conta_id: f.conta_pagamento_id, pagamentos: f.pagamentos
  };
}

function criarCompra_(c, k, o, id, demo) {
  const total = o.total, n = o.parcelas;
  const comp1 = competenciaCompra_(k, o.data_compra);
  const base = Math.floor(total / n), resto = total - base * n;
  const compra = marcarCriacao_({
    id: id, familia_id: c.familia.id, cartao_id: k.id, descricao: o.descricao, valor_total: paraTexto_(total), parcelas: String(n),
    valor_parcela: paraTexto_(base), categoria: o.categoria, subcategoria: o.subcategoria, data_compra: o.data_compra,
    responsavel: o.responsavel, observacoes: o.observacoes, primeira_fatura: comp1, demo: demo ? '1' : ''
  }, c.email);
  const parcelas = [], comps = [];
  for (let i = 0; i < n; i++) {
    const comp = addComp_(comp1, i);
    comps.push(comp);
    parcelas.push(marcarCriacao_({
      id: uuid_(), familia_id: c.familia.id, demo: demo ? '1' : '',
      descricao: (o.descricao + (n > 1 ? ' (' + (i + 1) + '/' + n + ')' : '')).slice(0, 130),
      valor: paraTexto_(base + (i === 0 ? resto : 0)), categoria: o.categoria, subcategoria: o.subcategoria,
      responsavel: o.responsavel, forma_pagamento: 'credito', conta_id: '', data_compra: addMesesDia_(o.data_compra, i),
      data_vencimento: datasFatura_(k, comp).vencimento, data_pagamento: '', status: 'pendente', recorrencia: 'nenhuma',
      parcela: n > 1 ? (i + 1) + '/' + n : '', observacoes: o.observacoes, anexo_url: '', valor_pago: '', pagamentos: '',
      serie_id: '', auto: '', grupo_id: '', cartao_id: k.id, compra_id: id, fatura: comp
    }, c.email));
  }
  inserirVarios_(tabela_('compras_cartao'), [compra]);
  inserirVarios_(tabela_('despesas'), parcelas);
  comps.forEach(function (comp) { sincronizarFatura_(c, k, comp); });
  return { compra: compra, primeiraFatura: comp1 };
}

function validarCompra_(r) {
  const o = {
    descricao: texto_(r.descricao, 'Descrição', 110, true),
    total: centavos_(r.valor_total, 'Valor'),
    parcelas: inteiro_(r.parcelas, 1, 1, 48, 'Parcelas'),
    categoria: texto_(r.categoria, 'Categoria', 60, true),
    subcategoria: texto_(r.subcategoria, 'Subcategoria', 60, false),
    data_compra: dataISO_(r.data_compra, 'Data da compra', true),
    responsavel: texto_(r.responsavel, 'Responsável', 60, true),
    observacoes: texto_(r.observacoes, 'Observações', 500, false)
  };
  if (o.total < o.parcelas) throw erro_('VALIDACAO', 'Valor muito baixo para o número de parcelas.');
  return o;
}

function acaoSalvarCompra_(d) {
  const c = ctx_();
  exigir_(c, 'criar');
  const r = d.compra || {};
  const id = String(r.id || '');
  if (!uuidValido_(id)) throw erro_('VALIDACAO', 'Identificador inválido.');
  const o = validarCompra_(r);
  return comLock_(function () {
    const existe = tabela_('compras_cartao').rows.find(function (x) { return x.id === id; });
    if (existe) {
      if (existe.familia_id !== c.familia.id) throw erro_('SEM_PERMISSAO', 'Registro não pertence à sua família.');
      return { compra: publico_(existe), repetido: true };
    }
    const k = cartaoDe_(r.cartao_id, c);
    if (k.status === 'inativo') throw erro_('VALIDACAO', 'Este cartão está inativo.');
    const res = criarCompra_(c, k, o, id, false);
    auditar_(c, 'criar', 'compras_cartao', id, { descricao: o.descricao, valor: paraTexto_(o.total), parcelas: o.parcelas, cartao: k.nome });
    marcarAlteracao_();
    return { compra: publico_(res.compra), primeiraFatura: res.primeiraFatura };
  });
}

/** Edita só descrição, categoria, responsável e observações. Valor, parcelas e data: excluir e lançar de novo. */
function acaoEditarCompra_(d) {
  const c = ctx_();
  exigir_(c, 'criar');
  const r = d.compra || {};
  const novo = {
    descricao: texto_(r.descricao, 'Descrição', 110, true),
    categoria: texto_(r.categoria, 'Categoria', 60, true),
    subcategoria: texto_(r.subcategoria, 'Subcategoria', 60, false),
    responsavel: texto_(r.responsavel, 'Responsável', 60, true),
    observacoes: texto_(r.observacoes, 'Observações', 500, false)
  };
  return comLock_(function () {
    const tc = tabela_('compras_cartao');
    const compra = tc.rows.find(function (x) { return x.id === r.id && x.familia_id === c.familia.id && x.excluido !== '1'; });
    if (!compra) throw erro_('NAO_ENCONTRADO', 'Compra não encontrada.');
    if (!podeAlterar_(c, compra)) throw erro_('SEM_PERMISSAO', 'Você não tem permissão para editar esta compra.');
    const mud = {};
    Object.keys(novo).forEach(function (k) { if ((compra[k] || '') !== novo[k]) mud[k] = [compra[k] || '', novo[k]]; });
    if (!Object.keys(mud).length) return { compra: publico_(compra), semMudancas: true };
    Object.assign(compra, novo, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(compra) });
    atualizar_(tc, compra);
    const td = tabela_('despesas');
    td.rows.forEach(function (p) {
      if (p.compra_id !== compra.id || p.familia_id !== c.familia.id) return;
      const n = parseInt(compra.parcelas, 10) || 1;
      Object.assign(p, {
        descricao: (novo.descricao + (n > 1 && p.parcela ? ' (' + p.parcela + ')' : '')).slice(0, 130),
        categoria: novo.categoria, subcategoria: novo.subcategoria, responsavel: novo.responsavel, observacoes: novo.observacoes,
        atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(p)
      });
      atualizar_(td, p);
    });
    auditar_(c, 'editar', 'compras_cartao', compra.id, mud);
    marcarAlteracao_();
    return { compra: publico_(compra) };
  });
}

function acaoExcluirCompra_(d, restaurar) {
  const c = ctx_();
  exigir_(c, 'criar');
  return comLock_(function () {
    const tc = tabela_('compras_cartao');
    const compra = tc.rows.find(function (x) { return x.id === d.id && x.familia_id === c.familia.id; });
    if (!compra) throw erro_('NAO_ENCONTRADO', 'Compra não encontrada.');
    if (!podeAlterar_(c, compra)) throw erro_('SEM_PERMISSAO', 'Você não tem permissão para excluir esta compra.');
    const alvo = restaurar ? '' : '1';
    if ((compra.excluido || '') === alvo) return { compra: publico_(compra) };
    const k = cartaoDe_(compra.cartao_id, c);
    Object.assign(compra, { excluido: alvo, atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(compra) });
    atualizar_(tc, compra);
    const td = tabela_('despesas'), comps = {};
    td.rows.forEach(function (p) {
      if (p.compra_id !== compra.id || p.familia_id !== c.familia.id) return;
      comps[p.fatura] = 1;
      Object.assign(p, { excluido: alvo, atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(p) });
      atualizar_(td, p);
    });
    Object.keys(comps).forEach(function (comp) { sincronizarFatura_(c, k, comp); });
    auditar_(c, restaurar ? 'restaurar' : 'excluir', 'compras_cartao', compra.id, { descricao: compra.descricao, valor: compra.valor_total });
    marcarAlteracao_();
    return { compra: publico_(compra) };
  });
}

function acaoCartoes_() {
  const c = ctx_();
  const fid = c.familia.id;
  const fats = faturasFamilia_(fid);
  const hoje = hoje_();
  const resumo = function (f) {
    return {
      cartao_id: f.cartao_id, competencia: f.competencia, fechamento: f.fechamento, vencimento: f.vencimento,
      total: f.total, pago: f.pago, restante: f.restante, status: f.status, pagamentos: f.pagamentos,
      parcelas: f.parcelas.map(publico_).sort(function (a, b) { return String(b.data_compra).localeCompare(String(a.data_compra)); })
    };
  };
  const cartoes = ativos_('cartoes', fid).map(function (k) {
    const fs = fats.filter(function (f) { return f.cartao_id === k.id; });
    const limite = cent_(k.limite);
    const usado = fs.reduce(function (s, f) { return s + Math.max(0, f.restante); }, 0);
    const compHoje = competenciaCompra_(k, hoje);
    const atual = fs.find(function (f) { return f.competencia === compHoje; });
    const aPagar = fs.find(function (f) { return f.restante > 0 && f.status !== 'aberta'; });
    return Object.assign(publico_(k), {
      limite_c: limite, usado: usado, disponivel: limite - usado,
      pct: limite > 0 ? Math.round(usado * 1000 / limite) / 10 : 0,
      competencia_atual: compHoje, datas_atual: datasFatura_(k, compHoje),
      fatura_atual: atual ? resumo(atual) : null,
      fatura_a_pagar: aPagar ? resumo(aPagar) : null,
      faturas: fs.map(resumo)
    });
  });
  const compras = ativos_('compras_cartao', fid).map(publico_)
    .sort(function (a, b) { return String(b.data_compra).localeCompare(String(a.data_compra)); });
  return { cartoes: cartoes, compras: compras, hoje: hoje };
}

// ------------------------------------------------------------
// Movimentações de dinheiro nas contas (base dos saldos)
// ------------------------------------------------------------

function movimentosReceita_(r) {
  if (r.status === 'cancelada') return [];
  const base = { tipo: 'receita', descricao: r.descricao, responsavel: r.responsavel, categoria: r.categoria, ref: r.id };
  const p = pags_(r);
  if (p.length) return p.map(function (x) { return Object.assign({ data: x.data, valor: cent_(x.valor), conta_id: x.conta_id || r.conta_id || '', pagamento_id: x.id }, base); });
  if (r.status === 'recebida') return [Object.assign({ data: r.data, valor: cent_(r.valor), conta_id: r.conta_id || '' }, base)];
  return [];
}

function movimentosDespesa_(r) {
  if (r.status === 'cancelada' || r.cartao_id) return [];
  const base = { tipo: 'despesa', descricao: r.descricao, responsavel: r.responsavel, categoria: r.categoria, ref: r.id };
  const p = pags_(r);
  if (p.length) return p.map(function (x) { return Object.assign({ data: x.data, valor: -cent_(x.valor), conta_id: x.conta_id || r.conta_id || '', pagamento_id: x.id }, base); });
  if (r.status === 'paga') return [Object.assign({ data: r.data_pagamento || r.data_compra, valor: -cent_(r.valor), conta_id: r.conta_id || '' }, base)];
  return [];
}

function movimentos_(fid) {
  const out = [];
  const push = function (m) { out.push(m); };
  ativos_('receitas', fid).forEach(function (r) { movimentosReceita_(r).forEach(push); });
  ativos_('despesas', fid).forEach(function (r) { movimentosDespesa_(r).forEach(push); });
  const cart = {};
  tabela_('cartoes').rows.forEach(function (k) { if (k.familia_id === fid) cart[k.id] = k; });
  ativos_('faturas', fid).forEach(function (f) {
    const k = cart[f.cartao_id];
    pags_(f).forEach(function (p) {
      out.push({
        data: p.data, valor: -cent_(p.valor), conta_id: p.conta_id || (k && k.conta_pagamento_id) || '', tipo: 'fatura',
        descricao: 'Pagamento da fatura ' + (k ? k.nome : '') + ' (' + mesAno_(f.competencia) + ')', responsavel: '',
        categoria: 'Cartão de crédito', ref: f.cartao_id + '|' + f.competencia, pagamento_id: p.id
      });
    });
  });
  ativos_('transferencias', fid).forEach(function (t) {
    const v = cent_(t.valor), desc = t.descricao || 'Transferência';
    out.push({ data: t.data, valor: -v, conta_id: t.conta_origem_id, tipo: 'transf', descricao: desc, responsavel: t.pessoa_origem, categoria: 'Transferência', ref: t.id });
    out.push({ data: t.data, valor: v, conta_id: t.conta_destino_id, tipo: 'transf', descricao: desc, responsavel: t.pessoa_destino, categoria: 'Transferência', ref: t.id });
  });
  return out;
}

/** Contas a pagar e a receber de um período (inclui sempre o que está atrasado). */
function acaoAgenda_(d) {
  const c = ctx_();
  const ini = dataISO_(d.inicio, 'Início', true), fim = dataISO_(d.fim, 'Fim', true);
  const fid = c.familia.id;
  const pagar = [], receber = [];
  ativos_('despesas', fid).forEach(function (r) {
    if (r.cartao_id || r.status === 'cancelada') return;
    const rest = restante_(r, 'despesas');
    if (!r.data_vencimento && rest <= 0) return; // gasto do dia a dia já pago não é "conta"
    const venc = r.data_vencimento || r.data_compra;
    const pagoEm = r.data_pagamento || '';
    if ((venc >= ini && venc <= fim) || (rest > 0 && venc < ini) || (rest <= 0 && pagoEm >= ini && pagoEm <= fim)) {
      pagar.push(Object.assign(publico_(r), { _tipo: 'despesa', _venc: venc, _restante: rest, _pago: quitado_(r, 'despesas') }));
    }
  });
  faturasFamilia_(fid).forEach(function (f) {
    if ((f.vencimento >= ini && f.vencimento <= fim) || (f.restante > 0 && f.vencimento < ini)) pagar.push(faturaItem_(f));
  });
  ativos_('receitas', fid).forEach(function (r) {
    if (r.status === 'cancelada') return;
    const rest = restante_(r, 'receitas'), p = pags_(r);
    const ultimo = p.length ? p.map(function (x) { return x.data; }).sort().pop() : '';
    if ((rest > 0 && r.data <= fim) || (p.length && ultimo >= ini && ultimo <= fim) || (r.auto === '1' && r.data >= ini && r.data <= fim)) {
      receber.push(Object.assign(publico_(r), { _tipo: 'receita', _venc: r.data, _restante: rest, _pago: quitado_(r, 'receitas') }));
    }
  });
  const ord = function (a, b) { return a._venc.localeCompare(b._venc); };
  return { pagar: pagar.sort(ord), receber: receber.sort(ord), hoje: hoje_() };
}

/** Saldos por conta, movimentações do período e saldo projetado. */
function acaoContas_(d) {
  const c = ctx_();
  const ini = dataISO_(d.inicio, 'Início', true), fim = dataISO_(d.fim, 'Fim', true);
  const fid = c.familia.id, hoje = hoje_();
  const ref = fim < hoje ? fim : hoje;
  const movs = movimentos_(fid);
  const fats = faturasFamilia_(fid);
  const rec = ativos_('receitas', fid), des = ativos_('despesas', fid);
  const contas = ativos_('contas_bancarias', fid);
  const ids = {};
  contas.forEach(function (k) { ids[k.id] = 1; });
  const lista = contas.map(function (k) {
    const base = k.data_saldo_inicial || '0000-01-01';
    const ms = movs.filter(function (m) { return m.conta_id === k.id && m.data >= base; });
    const ini0 = cent_(k.saldo_inicial);
    const saldoEm = function (x) { return (base <= x ? ini0 : 0) + ms.filter(function (m) { return m.data <= x; }).reduce(function (s, m) { return s + m.valor; }, 0); };
    const per = ms.filter(function (m) { return m.data >= ini && m.data <= fim; });
    const saldo = saldoEm(ref);
    let projetado = saldo;
    const previsto = { receber: 0, pagar: 0, faturas: 0, agendados: 0 };
    if (fim >= hoje) {
      ms.forEach(function (m) { if (m.data > hoje && m.data <= fim) previsto.agendados += m.valor; });
      rec.forEach(function (r) { if (r.conta_id === k.id && r.data <= fim) previsto.receber += restante_(r, 'receitas'); });
      des.forEach(function (r) { if (!r.cartao_id && r.conta_id === k.id && (r.data_vencimento || r.data_compra) <= fim) previsto.pagar += restante_(r, 'despesas'); });
      fats.forEach(function (f) { if (f.conta_pagamento_id === k.id && f.vencimento <= fim && f.restante > 0) previsto.faturas += f.restante; });
      projetado = saldo + previsto.agendados + previsto.receber - previsto.pagar - previsto.faturas;
    }
    return Object.assign(publico_(k), {
      saldo: saldo, projetado: projetado, previsto: previsto,
      entradas: per.reduce(function (s, m) { return s + (m.valor > 0 ? m.valor : 0); }, 0),
      saidas: per.reduce(function (s, m) { return s + (m.valor < 0 ? -m.valor : 0); }, 0),
      movimentos: per.sort(function (a, b) { return b.data.localeCompare(a.data); }).slice(0, 300)
    });
  });
  const semConta = movs.filter(function (m) { return !ids[m.conta_id] && m.tipo !== 'transf' && m.data >= ini && m.data <= fim; });
  return {
    contas: lista, ref: ref, hoje: hoje,
    total: lista.reduce(function (s, k) { return s + (k.status === 'inativa' ? 0 : k.saldo); }, 0),
    totalProjetado: lista.reduce(function (s, k) { return s + (k.status === 'inativa' ? 0 : k.projetado); }, 0),
    semConta: { qtd: semConta.length, valor: semConta.reduce(function (s, m) { return s + m.valor; }, 0) }
  };
}

// ------------------------------------------------------------
// Lançamentos recorrentes (gerados automaticamente, sem duplicar)
// ------------------------------------------------------------

function passo_(a, rec, k) {
  if (rec === 'semanal') return addDias_(a, 7 * k);
  if (rec === 'quinzenal') return addDias_(a, 14 * k);
  return addMesesDia_(a, ({ mensal: 1, trimestral: 3, anual: 12 }[rec] || 1) * k);
}

/**
 * Gera as próximas ocorrências de cada série até o fim do mês seguinte.
 * Nunca repete uma data que já existe na série (nem as que foram excluídas).
 */
function gerarRecorrencias_(c, so) {
  const fid = c.familia.id, hoje = hoje_();
  const Y = +hoje.slice(0, 4), M = +hoje.slice(5, 7);
  const p = addMes_(Y, M, 1);
  const horizonte = iso_(p.y, p.m, ultimoDia_(p.y, p.m));
  const piso = iso_(Y, M, 1);
  const fins = finsSerie_(fid);
  let total = 0;
  ['receitas', 'despesas'].forEach(function (ent) {
    if (so && so.ent && so.ent !== ent) return;
    const t = tabela_(ent);
    const campo = ent === 'receitas' ? 'data' : 'data_compra';
    const grupos = {};
    t.rows.forEach(function (r) {
      if (r.familia_id !== fid || !r.serie_id || r.cartao_id) return;
      if (so && so.serie && r.serie_id !== so.serie) return;
      (grupos[r.serie_id] = grupos[r.serie_id] || []).push(r);
    });
    const novos = [];
    Object.keys(grupos).forEach(function (sid) {
      const g = grupos[sid].slice().sort(function (a, b) { return String(a[campo]).localeCompare(String(b[campo])); });
      const ativos = g.filter(function (r) { return r.excluido !== '1'; });
      if (!ativos.length) return;
      const tpl = ativos[ativos.length - 1];
      if (RECORRENCIAS.indexOf(tpl.recorrencia) <= 0 || tpl.status === 'cancelada') return;
      const anchor = g[0][campo], ultima = g[g.length - 1][campo];
      const fim = fins[sid] && fins[sid] < horizonte ? fins[sid] : horizonte;
      const existentes = {};
      g.forEach(function (r) { existentes[r[campo]] = 1; });
      for (let k = 1; k <= 3000 && novos.length < 500; k++) {
        const dt = passo_(anchor, tpl.recorrencia, k);
        if (dt > fim) break;
        if (dt <= ultima || dt < piso || existentes[dt]) continue;
        existentes[dt] = 1;
        novos.push(ocorrencia_(ent, tpl, dt, sid, fid));
      }
    });
    if (novos.length) { inserirVarios_(t, novos); total += novos.length; }
  });
  return total;
}

function ocorrencia_(ent, tpl, dt, sid, fid) {
  const o = {
    id: uuid_(), familia_id: fid, descricao: tpl.descricao, valor: tpl.valor, categoria: tpl.categoria,
    responsavel: tpl.responsavel, conta_id: tpl.conta_id || '', recorrencia: tpl.recorrencia, observacoes: '', anexo_url: '',
    serie_id: sid, auto: '1', pagamentos: '', demo: tpl.demo || ''
  };
  if (ent === 'receitas') {
    Object.assign(o, { data: dt, status: 'prevista', valor_recebido: '' });
  } else {
    const off = tpl.data_vencimento ? Math.max(0, diasEntre_(tpl.data_compra, tpl.data_vencimento)) : 0;
    Object.assign(o, {
      subcategoria: tpl.subcategoria || '', forma_pagamento: tpl.forma_pagamento || '', data_compra: dt,
      data_vencimento: addDias_(dt, off), data_pagamento: '', status: 'pendente', parcela: '', valor_pago: '',
      grupo_id: '', cartao_id: '', compra_id: '', fatura: ''
    });
  }
  return marcarCriacao_(o, tpl.criado_por || 'sistema');
}

/** Roda uma vez por dia, na primeira abertura do app por um membro ou administrador. */
function rodarRecorrenciasDoDia_(c) {
  const props = PropertiesService.getScriptProperties();
  const hoje = hoje_();
  if (props.getProperty('RECORRENCIA_DIA') === hoje) return;
  const lock = LockService.getScriptLock();
  if (!lock.tryLock(5000)) return;
  try {
    limparCacheTabelas_();
    if (props.getProperty('RECORRENCIA_DIA') === hoje) return;
    const n = gerarRecorrencias_(c, null);
    props.setProperty('RECORRENCIA_DIA', hoje);
    if (n) { auditar_(c, 'gerar_recorrencias', 'sistema', '', { lancamentos: n }); marcarAlteracao_(); }
    SpreadsheetApp.flush();
  } catch (e) {
    console.error('Recorrências: ' + (e && e.message));
  } finally {
    lock.releaseLock();
    limparCacheTabelas_();
  }
}

/** Remove (exclusão lógica) as próximas ocorrências automáticas ainda em aberto de uma série. */
function encerrarFuturas_(c, ent, sid, aposData, exceto) {
  const t = tabela_(ent), campo = ent === 'receitas' ? 'data' : 'data_compra';
  let n = 0;
  t.rows.forEach(function (r) {
    if (r.familia_id !== c.familia.id || r.serie_id !== sid || r.id === exceto || r.excluido === '1') return;
    if (r.auto !== '1' || r[campo] <= aposData || pags_(r).length) return;
    if (r.status !== 'pendente' && r.status !== 'prevista') return;
    Object.assign(r, { excluido: '1', atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(r) });
    atualizar_(t, r);
    n++;
  });
  return n;
}

/** Leva as alterações de um lançamento recorrente para as próximas ocorrências em aberto. */
function aplicarProximas_(c, ent, novo, campo) {
  const t = tabela_(ent);
  const campos = ['descricao', 'valor', 'categoria', 'responsavel', 'conta_id'].concat(ent === 'despesas' ? ['subcategoria', 'forma_pagamento'] : []);
  let n = 0;
  t.rows.forEach(function (r) {
    if (r.familia_id !== c.familia.id || r.serie_id !== novo.serie_id || r.id === novo.id || r.excluido === '1') return;
    if (r[campo] <= novo[campo] || pags_(r).length || (r.status !== 'pendente' && r.status !== 'prevista')) return;
    if (!campos.some(function (k) { return String(r[k] || '') !== String(novo[k] || ''); })) return;
    campos.forEach(function (k) { r[k] = novo[k]; });
    Object.assign(r, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(r) });
    atualizar_(t, r);
    n++;
  });
  return n;
}

function finsSerie_(fid) {
  const o = {};
  tabela_('configuracoes').rows.forEach(function (r) {
    if (r.familia_id === fid && r.excluido !== '1' && String(r.chave).indexOf('serie_fim:') === 0) o[r.chave.slice(10)] = r.valor;
  });
  return o;
}

function setConfig_(c, chave, valor) {
  const t = tabela_('configuracoes');
  const r = t.rows.find(function (x) { return x.familia_id === c.familia.id && x.chave === chave && x.excluido !== '1'; });
  if (r) {
    Object.assign(r, { valor: valor, atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(r) });
    atualizar_(t, r);
  } else {
    inserirVarios_(t, [marcarCriacao_({ id: uuid_(), familia_id: c.familia.id, chave: chave, valor: valor }, c.email)]);
  }
}

// ============================================================
// 6. VALIDAÇÃO
// ============================================================

function validar_(ent, r, c) {
  r = r || {};
  if (ent === 'receitas') {
    return {
      descricao: texto_(r.descricao, 'Descrição', 120, true),
      valor: paraTexto_(centavos_(r.valor, 'Valor')),
      categoria: texto_(r.categoria, 'Categoria', 60, true),
      responsavel: texto_(r.responsavel, 'Responsável', 60, true),
      conta_id: contaValida_(r.conta_id, c),
      data: dataISO_(r.data, 'Data', true),
      status: opcao_(r.status, ['recebida', 'prevista', 'cancelada'], 'recebida', 'Status'),
      recorrencia: opcao_(r.recorrencia, RECORRENCIAS, 'nenhuma', 'Recorrência'),
      observacoes: texto_(r.observacoes, 'Observações', 500, false),
      anexo_url: urlOpcional_(r.anexo_url)
    };
  }
  if (ent === 'despesas') {
    const o = {
      descricao: texto_(r.descricao, 'Descrição', 120, true),
      valor: paraTexto_(centavos_(r.valor, 'Valor')),
      categoria: texto_(r.categoria, 'Categoria', 60, true),
      subcategoria: texto_(r.subcategoria, 'Subcategoria', 60, false),
      responsavel: texto_(r.responsavel, 'Responsável', 60, true),
      forma_pagamento: opcao_(r.forma_pagamento, FORMAS_PAGAMENTO, '', 'Forma de pagamento'),
      conta_id: contaValida_(r.conta_id, c),
      data_compra: dataISO_(r.data_compra, 'Data', true),
      data_vencimento: dataISO_(r.data_vencimento, 'Vencimento', false),
      data_pagamento: dataISO_(r.data_pagamento, 'Data de pagamento', false),
      status: opcao_(r.status, ['pendente', 'paga', 'cancelada'], 'pendente', 'Status'),
      recorrencia: opcao_(r.recorrencia, RECORRENCIAS, 'nenhuma', 'Recorrência'),
      parcela: texto_(r.parcela, 'Parcela', 10, false),
      observacoes: texto_(r.observacoes, 'Observações', 500, false),
      anexo_url: urlOpcional_(r.anexo_url)
    };
    if (o.status === 'paga' && !o.data_pagamento) o.data_pagamento = o.data_compra;
    if (o.status !== 'paga') o.data_pagamento = '';
    return o;
  }
  if (ent === 'contas_bancarias') return validarConta_(r);
  if (ent === 'cartoes') {
    return {
      nome: texto_(r.nome, 'Nome do cartão', 60, true),
      instituicao: texto_(r.instituicao, 'Banco ou instituição', 60, false),
      titular: texto_(r.titular, 'Titular', 60, false),
      limite: paraTexto_(centavos_(r.limite, 'Limite total')),
      dia_fechamento: String(inteiro_(r.dia_fechamento, null, 1, 31, 'Dia de fechamento')),
      dia_vencimento: String(inteiro_(r.dia_vencimento, null, 1, 31, 'Dia de vencimento')),
      conta_pagamento_id: contaValida_(r.conta_pagamento_id, c),
      cor: /^#[0-9a-f]{6}$/i.test(r.cor || '') ? r.cor : '#7c3aed',
      status: opcao_(r.status, ['ativo', 'inativo'], 'ativo', 'Status')
    };
  }
  if (ent === 'orcamentos') {
    return {
      competencia: competenciaValida_(r.competencia),
      categoria: texto_(r.categoria, 'Categoria', 60, true),
      subcategoria: texto_(r.subcategoria, 'Subcategoria', 60, false),
      valor_planejado: paraTexto_(centavos_(r.valor_planejado, 'Valor planejado'))
    };
  }
  if (ent === 'metas') {
    const o = {
      nome: texto_(r.nome, 'Nome da meta', 80, true),
      tipo: opcao_(r.tipo, ['reserva', 'imovel', 'reforma', 'viagem', 'veiculo', 'dividas', 'investimentos', 'outro'], 'outro', 'Tipo'),
      valor_desejado: paraTexto_(centavos_(r.valor_desejado, 'Valor desejado')),
      valor_inicial: paraTexto_(centavos_((r.valor_inicial === '' || r.valor_inicial == null) ? '0' : r.valor_inicial, 'Valor já guardado', { zero: true })),
      data_inicio: dataISO_(r.data_inicio, 'Data de início', false) || hoje_(),
      prazo: dataISO_(r.prazo, 'Prazo', false),
      responsavel: texto_(r.responsavel, 'Responsável', 60, false),
      prioridade: opcao_(r.prioridade, ['alta', 'media', 'baixa'], 'media', 'Prioridade'),
      descricao: texto_(r.descricao, 'Descrição', 500, false),
      status: opcao_(r.status, ['ativa', 'concluida', 'pausada', 'cancelada'], 'ativa', 'Situação'),
      conta_id: contaValida_(r.conta_id, c)
    };
    if (o.prazo && o.prazo < o.data_inicio) throw erro_('VALIDACAO', 'O prazo deve ser depois da data de início.');
    return o;
  }
  if (ent === 'dividas') {
    const o = {
      nome: texto_(r.nome, 'Nome da dívida', 80, true),
      credor: texto_(r.credor, 'Instituição ou credor', 60, false),
      tipo: opcao_(r.tipo, ['emprestimo', 'financiamento', 'cartao', 'pessoa', 'outro'], 'emprestimo', 'Tipo'),
      valor_original: paraTexto_(centavos_(r.valor_original, 'Valor original')),
      saldo_inicial: paraTexto_(centavos_(r.saldo_inicial, 'Saldo devedor atual', { zero: true })),
      taxa_juros: taxa_(r.taxa_juros),
      qtd_parcelas: String(inteiro_(r.qtd_parcelas, null, 1, 600, 'Quantidade de parcelas')),
      valor_parcela_inicial: paraTexto_(centavos_(r.valor_parcela_inicial, 'Valor da parcela')),
      parcelas_pagas_inicial: String(inteiro_(r.parcelas_pagas_inicial, 0, 0, 600, 'Parcelas já pagas')),
      data_contratacao: dataISO_(r.data_contratacao, 'Data de contratação', true),
      dia_vencimento: String(inteiro_(r.dia_vencimento, null, 1, 31, 'Dia do vencimento')),
      conta_id: contaValida_(r.conta_id, c),
      status: opcao_(r.status, ['ativa', 'quitada', 'cancelada'], 'ativa', 'Situação')
    };
    if (+o.parcelas_pagas_inicial > +o.qtd_parcelas) throw erro_('VALIDACAO', 'Parcelas já pagas não pode ser maior que o total de parcelas.');
    if (cent_(o.saldo_inicial) > cent_(o.valor_original) * 3) throw erro_('VALIDACAO', 'Confira o saldo devedor: está muito acima do valor original.');
    return o;
  }
  if (ent === 'transferencias') {
    const o = {
      pessoa_origem: texto_(r.pessoa_origem, 'Pessoa de origem', 60, false),
      pessoa_destino: texto_(r.pessoa_destino, 'Pessoa de destino', 60, false),
      conta_origem_id: contaValida_(r.conta_origem_id, c),
      conta_destino_id: contaValida_(r.conta_destino_id, c),
      valor: paraTexto_(centavos_(r.valor, 'Valor')),
      data: dataISO_(r.data, 'Data', true),
      descricao: texto_(r.descricao, 'Descrição', 120, false)
    };
    if (!o.conta_origem_id || !o.conta_destino_id) throw erro_('VALIDACAO', 'Escolha a conta de origem e a de destino.');
    if (o.conta_origem_id === o.conta_destino_id) throw erro_('VALIDACAO', 'A conta de origem e a de destino devem ser diferentes.');
    return o;
  }
  if (ent === 'categorias') {
    return {
      tipo: opcao_(r.tipo, ['receita', 'despesa'], null, 'Tipo'),
      grupo: texto_(r.grupo, 'Grupo', 60, false),
      nome: texto_(r.nome, 'Nome', 60, true),
      cor: /^#[0-9a-f]{6}$/i.test(r.cor || '') ? r.cor : '',
      icone: texto_(r.icone, 'Ícone', 8, false),
      ativo: (r.ativo === false || r.ativo === '0') ? '0' : '1'
    };
  }
  throw erro_('ACAO_INVALIDA', 'Tipo de registro inválido.');
}

function validarConta_(r) {
  const saldo = (r.saldo_inicial === '' || r.saldo_inicial == null) ? '0' : r.saldo_inicial;
  return {
    nome: texto_(r.nome, 'Nome da conta', 60, true),
    instituicao: texto_(r.instituicao, 'Instituição', 60, false),
    titular: texto_(r.titular, 'Titular', 60, false),
    tipo: opcao_(r.tipo, TIPOS_CONTA, 'corrente', 'Tipo de conta'),
    saldo_inicial: paraTexto_(centavos_(saldo, 'Saldo inicial', { negativo: true, zero: true })),
    data_saldo_inicial: dataISO_(r.data_saldo_inicial, 'Data do saldo inicial', false) || hoje_(),
    status: opcao_(r.status, ['ativa', 'inativa'], 'ativa', 'Status')
  };
}

function contaValida_(id, c) {
  id = String(id || '').trim();
  if (!id) return '';
  const ok = tabela_('contas_bancarias').rows.some(function (r) { return r.id === id && r.familia_id === c.familia.id && r.excluido !== '1'; });
  if (!ok) throw erro_('VALIDACAO', 'Conta bancária inválida.');
  return id;
}

function texto_(v, rotulo, max, obrigatorio) {
  const s = String(v == null ? '' : v).replace(/[\u0000-\u001f\u007f]/g, ' ').trim();
  if (!s && obrigatorio) throw erro_('VALIDACAO', rotulo + ' é obrigatório.');
  if (s.length > max) throw erro_('VALIDACAO', rotulo + ' deve ter no máximo ' + max + ' caracteres.');
  return s;
}

function inteiro_(v, padrao, min, max, rotulo) {
  const s = String(v == null ? '' : v).trim();
  if (s === '' && padrao !== null && padrao !== undefined) return padrao;
  if (!/^\d+$/.test(s)) throw erro_('VALIDACAO', rotulo + ' inválido.');
  const n = parseInt(s, 10);
  if (n < min || n > max) throw erro_('VALIDACAO', rotulo + ' deve estar entre ' + min + ' e ' + max + '.');
  return n;
}

function uuidValido_(id) {
  return /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(String(id || ''));
}

function opcao_(v, lista, padrao, rotulo) {
  const s = String(v == null ? '' : v).trim();
  if (s === '' && padrao !== null && padrao !== undefined) return padrao;
  if (lista.indexOf(s) < 0) throw erro_('VALIDACAO', rotulo + ' inválido.');
  return s;
}

function urlOpcional_(v) {
  const s = String(v || '').trim();
  if (!s) return '';
  if (!/^https:\/\/[^\s]+$/i.test(s) || s.length > 500) throw erro_('VALIDACAO', 'Link do comprovante inválido.');
  return s;
}

/** Converte um valor em centavos (inteiro). Aceita 1234.56, "1.234,56", "R$ 10,00". */
function centavos_(v, rotulo, opt) {
  opt = opt || {};
  let n;
  if (typeof v === 'number') {
    n = v;
  } else {
    let s = String(v == null ? '' : v).replace(/R\$|\s/g, '');
    if (!s) throw erro_('VALIDACAO', rotulo + ' é obrigatório.');
    if (s.indexOf(',') >= 0) s = s.replace(/\./g, '').replace(',', '.');
    else if (/^-?\d{1,3}(\.\d{3})+$/.test(s)) s = s.replace(/\./g, '');
    if (!/^-?\d+(\.\d+)?$/.test(s)) throw erro_('VALIDACAO', rotulo + ' inválido.');
    n = Number(s);
  }
  if (!isFinite(n)) throw erro_('VALIDACAO', rotulo + ' inválido.');
  const c = Math.round(n * 100);
  if (!opt.negativo && c < 0) throw erro_('VALIDACAO', rotulo + ' não pode ser negativo.');
  if (!opt.zero && c === 0) throw erro_('VALIDACAO', rotulo + ' deve ser maior que zero.');
  if (Math.abs(c) > 99999999999) throw erro_('VALIDACAO', rotulo + ' muito alto.');
  return c;
}

/** Leitura tolerante de valores já gravados (retorna centavos; inválido = 0). */
function cent_(s) {
  if (s === '' || s == null) return 0;
  try { return centavos_(s, 'valor', { negativo: true, zero: true }); } catch (e) { return 0; }
}

function paraTexto_(c) {
  const neg = c < 0;
  c = Math.abs(c);
  return (neg ? '-' : '') + Math.floor(c / 100) + '.' + String(c % 100).padStart(2, '0');
}

function brl_(c) {
  const neg = c < 0;
  c = Math.abs(Math.round(c));
  const inteiro = String(Math.floor(c / 100)).replace(/\B(?=(\d{3})+(?!\d))/g, '.');
  return (neg ? '-' : '') + 'R$ ' + inteiro + ',' + String(c % 100).padStart(2, '0');
}

function dataISO_(v, rotulo, obrigatorio) {
  let s = norm_(v);
  if (!s) {
    if (obrigatorio) throw erro_('VALIDACAO', rotulo + ' é obrigatória.');
    return '';
  }
  let m = s.match(/^(\d{2})\/(\d{2})\/(\d{4})$/);
  if (m) s = m[3] + '-' + m[2] + '-' + m[1];
  m = s.match(/^(\d{4})-(\d{2})-(\d{2})/);
  if (!m) throw erro_('VALIDACAO', rotulo + ' inválida.');
  const y = +m[1], mo = +m[2], d = +m[3];
  const dt = new Date(Date.UTC(y, mo - 1, d));
  if (dt.getUTCMonth() !== mo - 1 || dt.getUTCDate() !== d || y < 1990 || y > 2100) throw erro_('VALIDACAO', rotulo + ' inválida.');
  return m[1] + '-' + m[2] + '-' + m[3];
}

// ============================================================
// 7. DASHBOARD — regras de cálculo transparentes (em centavos)
// ============================================================

function acaoDashboard_(d) {
  const c = ctx_();
  const ini = dataISO_(d.inicio, 'Início', true);
  const fim = dataISO_(d.fim, 'Fim', true);
  if (ini > fim) throw erro_('VALIDACAO', 'A data inicial deve ser anterior à final.');
  if (diasEntre_(ini, fim) > 3700) throw erro_('VALIDACAO', 'Período muito longo (máximo 10 anos).');
  const f = d.filtros || {};
  const hoje = hoje_();
  const fid = c.familia.id;

  let contas = ativos_('contas_bancarias', fid);
  if (f.conta_id) contas = contas.filter(function (x) { return x.id === f.conta_id; });
  const passa = function (r) {
    return (!f.responsavel || r.responsavel === f.responsavel) &&
      (!f.categoria || r.categoria === f.categoria) &&
      (!f.conta_id || r.conta_id === f.conta_id);
  };
  const rec = ativos_('receitas', fid).filter(function (r) { return r.status !== 'cancelada' && passa(r); });
  const des = ativos_('despesas', fid).filter(function (r) { return r.status !== 'cancelada' && passa(r); });

  // Filtros por pessoa/categoria não se aplicam a saldos de conta → mostramos o movimento líquido.
  const parcial = !!(f.responsavel || f.categoria);
  const todasFaturas = faturasFamilia_(fid);
  const fats = parcial ? [] : todasFaturas.filter(function (x) { return !f.conta_id || x.conta_pagamento_id === f.conta_id; });

  let movs = [];
  if (parcial) {
    rec.forEach(function (r) { movimentosReceita_(r).forEach(function (m) { movs.push(m); }); });
    des.forEach(function (r) {
      if (r.cartao_id) {
        if (r.status === 'paga' && r.data_pagamento) movs.push({ data: r.data_pagamento, valor: -cent_(r.valor), conta_id: '', tipo: 'despesa', descricao: r.descricao, responsavel: r.responsavel, categoria: r.categoria, ref: r.id });
      } else movimentosDespesa_(r).forEach(function (m) { movs.push(m); });
    });
  } else {
    movs = movimentos_(fid);
  }
  if (f.conta_id) movs = movs.filter(function (m) { return m.conta_id === f.conta_id; });

  const V = function (r) { return cent_(r.valor); };
  const soma = function (arr) { return arr.reduce(function (s, r) { return s + V(r); }, 0); };
  const somaQ = function (arr, ent) { return arr.reduce(function (s, r) { return s + quitado_(r, ent); }, 0); };
  const competencia = function (r) { return r.data_compra || r.data_vencimento || ''; };
  const vencimento = function (r) { return r.data_vencimento || r.data_compra || ''; };
  const recEm = function (a, b) { return rec.filter(function (r) { return r.data >= a && r.data <= b; }); };
  const desEm = function (a, b) { return des.filter(function (r) { const x = competencia(r); return x >= a && x <= b; }); };

  const inicioConta = {};
  contas.forEach(function (x) { inicioConta[x.id] = x.data_saldo_inicial || '0000-01-01'; });
  const menorInicio = contas.map(function (x) { return inicioConta[x.id]; }).sort()[0] || '0000-01-01';
  const baseDe = function (m) { return (m.conta_id && inicioConta[m.conta_id]) ? inicioConta[m.conta_id] : menorInicio; };

  /** Saldo = saldo inicial + entradas − saídas ± transferências (que se anulam no total da família). */
  function composicaoSaldo(ref) {
    let inicial = 0, entradas = 0, saidas = 0, tr = 0;
    if (!parcial) contas.forEach(function (x) { if ((x.data_saldo_inicial || '') <= ref) inicial += cent_(x.saldo_inicial); });
    movs.forEach(function (m) {
      if (m.data > ref || m.data < baseDe(m)) return;
      if (m.tipo === 'transf') tr += m.valor;
      else if (m.valor > 0) entradas += m.valor;
      else saidas -= m.valor;
    });
    return { inicial: inicial, recebidas: entradas, pagas: saidas, transferencias: tr, saldo: inicial + entradas - saidas + tr };
  }

  const itemDespesa = function (r) { return Object.assign(publico_(r), { _tipo: 'despesa', _venc: vencimento(r), _restante: restante_(r, 'despesas') }); };
  const rP = recEm(ini, fim), dP = desEm(ini, fim);
  const recTotal = soma(rP);
  const recRecebido = somaQ(rP, 'receitas');
  const desTotal = soma(dP);
  const desPagoComp = somaQ(dP, 'despesas');
  const ref = fim < hoje ? fim : hoje;
  const saldo = composicaoSaldo(ref);

  // Contas em aberto (despesas fora do cartão + faturas) com vencimento até o fim do período.
  const abertos = des.filter(function (r) { return !r.cartao_id && restante_(r, 'despesas') > 0; }).map(itemDespesa)
    .concat(fats.filter(function (x) { return x.restante > 0; }).map(faturaItem_))
    .sort(function (a, b) { return a._venc.localeCompare(b._venc); });
  const pendentes = abertos.filter(function (x) { return x._venc <= fim; });
  const somaR = function (arr) { return arr.reduce(function (s, x) { return s + x._restante; }, 0); };
  const vencidas = pendentes.filter(function (x) { return x._venc < hoje; });

  // Pagamentos feitos no período (saídas de dinheiro, sem contar transferências).
  const saidasPer = movs.filter(function (m) { return m.valor < 0 && m.tipo !== 'transf' && m.data >= ini && m.data <= fim; });
  const totalPagas = -saidasPer.reduce(function (s, m) { return s + m.valor; }, 0);
  const dividas = ativos_('dividas', fid).filter(function (x) { return x.status !== 'quitada' && x.status !== 'cancelada'; });
  const totalDividas = dividas.reduce(function (s, x) { return s + cent_(x.saldo_devedor); }, 0);
  const economia = recRecebido - desTotal;
  const baseComp = c.familia.base_comprometimento === 'pagas' ? totalPagas : desTotal;
  const comprometido = recTotal > 0 ? Math.round(baseComp * 1000 / recTotal) / 10 : null;

  // ---- séries mensais ----
  const yi = +ini.slice(0, 4), mi = +ini.slice(5, 7), yf = +fim.slice(0, 4), mf = +fim.slice(5, 7);
  const nMeses = (yf * 12 + mf) - (yi * 12 + mi) + 1;
  const qtd = nMeses <= 1 ? 6 : Math.min(nMeses, 24);
  const comeco = nMeses <= 1 ? addMes_(yf, mf, -5) : (nMeses > 24 ? addMes_(yf, mf, -23) : { y: yi, m: mi });
  const serie = [];
  let acumulado = 0;
  for (let i = 0; i < qtd; i++) {
    const p = addMes_(comeco.y, comeco.m, i);
    const a = iso_(p.y, p.m, 1), b = iso_(p.y, p.m, ultimoDia_(p.y, p.m));
    const r = recEm(a, b), dd = desEm(a, b);
    const recebidas = somaQ(r, 'receitas');
    const futuro = a > hoje;
    const eco = futuro ? null : recebidas - soma(dd);
    if (eco !== null) acumulado += eco;
    serie.push({
      mes: p.y + '-' + String(p.m).padStart(2, '0'),
      receitas: soma(r), recebidas: recebidas, despesas: soma(dd),
      economia: eco, economiaAcumulada: futuro ? null : acumulado,
      saldo: futuro ? null : composicaoSaldo(b < hoje ? b : hoje).saldo
    });
  }

  const agrupar = function (arr, campo) {
    const m = {};
    arr.forEach(function (r) { const k = r[campo] || 'Sem ' + campo; m[k] = (m[k] || 0) + V(r); });
    return Object.keys(m).map(function (k) { return { nome: k, valor: m[k] }; }).sort(function (a, b) { return b.valor - a.valor; });
  };

  // ---- comparativo com o período anterior de mesmo tamanho ----
  const mesesInteiros = ini.slice(8) === '01' && fim === iso_(yf, mf, ultimoDia_(yf, mf));
  let aIni, aFim;
  if (mesesInteiros) {
    const p0 = addMes_(yi, mi, -nMeses), p1 = addMes_(yf, mf, -nMeses);
    aIni = iso_(p0.y, p0.m, 1); aFim = iso_(p1.y, p1.m, ultimoDia_(p1.y, p1.m));
  } else {
    const len = diasEntre_(ini, fim) + 1;
    aIni = addDias_(ini, -len); aFim = addDias_(ini, -1);
  }
  const rA = recEm(aIni, aFim), dA = desEm(aIni, aFim);
  const anterior = { receitas: soma(rA), despesas: soma(dA), economia: somaQ(rA, 'receitas') - soma(dA), inicio: aIni, fim: aFim };

  // ---- mensagens automáticas (regras simples, com valores) ----
  const insights = [];
  const umMes = nMeses === 1 && mesesInteiros;
  if (umMes && !parcial && !f.conta_id) {
    const orc = orcamentoMes_(fid, ini.slice(0, 7));
    if (orc.totalPlanejado > 0) {
      const gastoOrc = orc.totalGastoOrcado;
      const pct = Math.round(gastoOrc * 100 / orc.totalPlanejado);
      if (gastoOrc <= orc.totalPlanejado) insights.push({ tipo: 'ok', acao: 'orcamento', texto: 'Suas despesas estão dentro do orçamento: ' + brl_(gastoOrc) + ' de ' + brl_(orc.totalPlanejado) + ' planejados (' + pct + '%).' });
      else insights.push({ tipo: 'perigo', acao: 'orcamento', texto: 'As despesas ultrapassaram o orçamento em ' + brl_(gastoOrc - orc.totalPlanejado) + ' (' + brl_(gastoOrc) + ' de ' + brl_(orc.totalPlanejado) + ').' });
      orc.linhas.filter(function (l) { return l.nivel >= 70; }).sort(function (a, b) { return b.pct - a.pct; }).slice(0, 3).forEach(function (l) {
        const nome = l.subcategoria ? l.categoria + ' ▸ ' + l.subcategoria : l.categoria;
        const txt = l.nivel > 100 ? 'Orçamento de ' + nome + ' ultrapassado em ' + brl_(l.gasto - l.planejado) + ' (' + brl_(l.gasto) + ' de ' + brl_(l.planejado) + ').'
          : l.nivel === 100 ? 'Orçamento de ' + nome + ' atingido: ' + brl_(l.gasto) + ' de ' + brl_(l.planejado) + '.'
          : 'Orçamento de ' + nome + ' com ' + fmtPct_(l.pct) + ' usado (' + brl_(l.gasto) + ' de ' + brl_(l.planejado) + '), acima do alerta de ' + l.nivel + '%.';
        insights.push({ tipo: l.nivel >= 100 ? 'perigo' : 'alerta', acao: 'orcamento', texto: txt });
      });
    }
  }
  if (!parcial && !f.conta_id) {
    ativos_('metas', fid).forEach(function (m) {
      const info = infoMeta_(m, hoje);
      if (m.status === 'concluida' && m.atualizado_em && m.atualizado_em.slice(0, 10) >= addDias_(hoje, -30)) {
        insights.push({ tipo: 'ok', acao: 'metas', texto: 'Meta alcançada: ' + m.nome + ' (' + brl_(info.acumulado) + ' de ' + brl_(info.desejado) + ').' });
      } else if (m.status === 'ativa' && m.prazo && info.dias >= 0 && info.dias <= 30) {
        insights.push({ tipo: 'alerta', acao: 'metas', texto: 'Prazo da meta ' + m.nome + ' termina em ' + plural_(info.dias, 'dia', 'dias') + ' — faltam ' + brl_(info.restante) + '.' });
      }
    });
  }
  const limite7 = addDias_(hoje, 7);
  const proximas = abertos.filter(function (x) { return x._venc >= hoje && x._venc <= limite7; });
  const todasVencidas = abertos.filter(function (x) { return x._venc < hoje; });
  if (proximas.length) insights.push({ tipo: 'alerta', acao: 'proximas', texto: 'Você possui ' + plural_(proximas.length, 'conta', 'contas') + ' vencendo nos próximos 7 dias, somando ' + brl_(somaR(proximas)) + '.' });
  if (todasVencidas.length) insights.push({ tipo: 'perigo', acao: 'vencidas', texto: plural_(todasVencidas.length, 'conta vencida', 'contas vencidas') + ' e ainda não ' + (todasVencidas.length > 1 ? 'pagas' : 'paga') + ', somando ' + brl_(somaR(todasVencidas)) + '.' });
  if (!parcial && !f.conta_id) {
    ativos_('cartoes', fid).forEach(function (k) {
      const limite = cent_(k.limite);
      if (k.status === 'inativo' || limite <= 0) return;
      const usado = todasFaturas.filter(function (x) { return x.cartao_id === k.id; }).reduce(function (s, x) { return s + Math.max(0, x.restante); }, 0);
      const pct = Math.round(usado * 100 / limite);
      const faixa = pct >= 95 ? 95 : pct >= 85 ? 85 : pct >= 70 ? 70 : 0;
      if (faixa) insights.push({ tipo: faixa >= 95 ? 'perigo' : 'alerta', acao: 'cartoes', texto: 'Cartão ' + k.nome + ': ' + pct + '% do limite usado (' + brl_(usado) + ' de ' + brl_(limite) + '), acima do alerta de ' + faixa + '%.' });
    });
  }
  if (anterior.despesas > 0 && desTotal > 0) {
    const varPct = Math.round((desTotal - anterior.despesas) * 1000 / anterior.despesas) / 10;
    if (varPct >= 5) insights.push({ tipo: 'alerta', texto: 'Seus gastos aumentaram ' + fmtPct_(varPct) + ' em relação ao período anterior (' + brl_(anterior.despesas) + ' → ' + brl_(desTotal) + ').' });
    else if (varPct <= -5) insights.push({ tipo: 'ok', texto: 'Seus gastos diminuíram ' + fmtPct_(-varPct) + ' em relação ao período anterior (' + brl_(anterior.despesas) + ' → ' + brl_(desTotal) + ').' });
  }
  const meta = cent_(c.familia.meta_economia_mensal);
  if (umMes && meta > 0) {
    if (economia >= meta) insights.push({ tipo: 'ok', texto: 'Você atingiu sua meta de economia deste mês: ' + brl_(economia) + ' economizados (meta de ' + brl_(meta) + ').' });
    else insights.push({ tipo: 'info', texto: 'Faltam ' + brl_(meta - economia) + ' para a meta de economia do mês (' + brl_(Math.max(economia, 0)) + ' de ' + brl_(meta) + ').' });
  }
  if (recTotal > 0 && desTotal > recTotal) {
    insights.push({ tipo: 'perigo', texto: 'As despesas do período (' + brl_(desTotal) + ') superam as receitas (' + brl_(recTotal) + ') em ' + brl_(desTotal - recTotal) + '.' });
  }
  if (!rec.length && !des.length) {
    insights.push({ tipo: 'info', texto: 'Nenhum lançamento ainda. Toque no botão + para registrar a primeira receita ou despesa.' });
  }

  const pessoas = {};
  membros_(fid).forEach(function (m) { if (m.status !== 'revogado') pessoas[m.nome] = 1; });
  ativos_('receitas', fid).concat(ativos_('despesas', fid)).forEach(function (r) { if (r.responsavel) pessoas[r.responsavel] = 1; });

  const ordenar = function (arr, campo) {
    return arr.slice().sort(function (a, b) { return String(b[campo] || '').localeCompare(String(a[campo] || '')); }).map(publico_);
  };

  return {
    periodo: { inicio: ini, fim: fim, ref: ref, hoje: hoje, meses: nMeses },
    parcial: parcial,
    cards: {
      receitas: { total: recTotal, recebido: recRecebido, previsto: recTotal - recRecebido },
      despesas: { total: desTotal, pago: desPagoComp, pendente: desTotal - desPagoComp },
      saldo: saldo,
      pendentes: { total: somaR(pendentes), qtd: pendentes.length, vencidas: vencidas.length, totalVencidas: somaR(vencidas) },
      pagas: { total: totalPagas, qtd: saidasPer.length },
      dividas: { total: totalDividas, qtd: dividas.length },
      economia: { total: economia, recebido: recRecebido, despesas: desTotal },
      comprometido: { pct: comprometido, base: c.familia.base_comprometimento === 'pagas' ? 'pagas' : 'assumidas', despesas: baseComp, receitas: recTotal }
    },
    serie: serie,
    porCategoria: agrupar(dP, 'categoria'),
    porPessoa: agrupar(dP, 'responsavel'),
    comparativo: { atual: { receitas: recTotal, despesas: desTotal, economia: economia }, anterior: anterior },
    insights: insights,
    detalhes: {
      receitas: ordenar(rP, 'data'),
      despesas: ordenar(dP, 'data_compra'),
      pendentes: pendentes,
      pagas: saidasPer.slice().sort(function (a, b) { return b.data.localeCompare(a.data); }).map(function (m) { return Object.assign({ _tipo: 'mov' }, m); }),
      proximas: proximas,
      vencidas: todasVencidas
    },
    opcoes: { pessoas: Object.keys(pessoas).sort() }
  };
}

// ============================================================
// 7b. PLANEJAMENTO — orçamento, metas e dívidas (Etapa 3)
// ============================================================

/** Campos calculados que dependem do histórico (metas e dívidas). */
function calcular_(ent, r) {
  if (ent === 'metas') {
    const info = infoMeta_(r, hoje_());
    r.valor_acumulado = paraTexto_(info.acumulado);
    if (r.status === 'ativa' || r.status === 'concluida') r.status = info.acumulado >= info.desejado ? 'concluida' : 'ativa';
  }
  if (ent === 'dividas') {
    const x = calcDivida_(r);
    r.saldo_devedor = paraTexto_(x.saldo);
    r.valor_parcela = paraTexto_(x.parcela);
    r.parcelas_pagas = String(x.pagas);
    r.parcelas_restantes = String(x.restantes);
    if (r.status !== 'cancelada') r.status = x.saldo <= 0 ? 'quitada' : 'ativa';
  }
}

function taxa_(v) {
  const s = String(v == null ? '' : v).replace('%', '').replace(',', '.').trim();
  if (!s) return '0';
  if (!/^\d+(\.\d+)?$/.test(s) || +s > 30) throw erro_('VALIDACAO', 'Taxa de juros inválida (informe ao mês, entre 0 e 30%).');
  return String(+s);
}

function movs_(r) {
  if (!r || !r.movimentos) return [];
  try { const a = JSON.parse(r.movimentos); return Array.isArray(a) ? a : []; } catch (e) { return []; }
}

function orcamentoUnico_(c, o, id) {
  const dup = tabela_('orcamentos').rows.some(function (x) {
    return x.id !== id && x.familia_id === c.familia.id && x.excluido !== '1' && x.competencia === o.competencia &&
      x.categoria === o.categoria && (x.subcategoria || '') === (o.subcategoria || '');
  });
  if (dup) throw erro_('VALIDACAO', 'Já existe um orçamento para esta categoria neste mês. Edite o existente.');
}

// ---------------- Orçamento ----------------

/** Planejado × gasto de um mês. Gasto = despesas do mês (pela data da compra), incluindo parcelas de cartão. */
function orcamentoMes_(fid, comp) {
  const ini = comp + '-01', fim = iso_(+comp.slice(0, 4), +comp.slice(5, 7), ultimoDia_(+comp.slice(0, 4), +comp.slice(5, 7)));
  const des = ativos_('despesas', fid).filter(function (r) { return r.status !== 'cancelada' && r.data_compra >= ini && r.data_compra <= fim; });
  const porCat = {}, porSub = {};
  let totalMes = 0;
  des.forEach(function (r) {
    const v = cent_(r.valor);
    totalMes += v;
    porCat[r.categoria] = (porCat[r.categoria] || 0) + v;
    const k = r.categoria + '||' + (r.subcategoria || '');
    porSub[k] = (porSub[k] || 0) + v;
  });
  const linhasOrc = ativos_('orcamentos', fid).filter(function (o) { return o.competencia === comp; });
  const temCat = {};
  linhasOrc.forEach(function (o) { if (!o.subcategoria) temCat[o.categoria] = 1; });
  let totalPlanejado = 0;
  const catsOrcadas = {};
  const linhas = linhasOrc.map(function (o) {
    const plan = cent_(o.valor_planejado);
    const gasto = o.subcategoria ? (porSub[o.categoria + '||' + o.subcategoria] || 0) : (porCat[o.categoria] || 0);
    if (!o.subcategoria || !temCat[o.categoria]) { totalPlanejado += plan; catsOrcadas[o.categoria] = 1; }
    const pct = plan > 0 ? Math.round(gasto * 1000 / plan) / 10 : 0;
    const nivel = gasto > plan ? 101 : gasto === plan && plan > 0 ? 100 : pct >= 85 ? 85 : pct >= 70 ? 70 : 0;
    return Object.assign(publico_(o), { planejado: plan, gasto: gasto, disponivel: plan - gasto, pct: pct, nivel: nivel });
  }).sort(function (a, b) { return (a.categoria + (a.subcategoria || '')).localeCompare(b.categoria + (b.subcategoria || '')); });
  // Gasto das categorias que têm orçamento (sem contar duas vezes categoria e subcategoria).
  let totalGastoOrcado = 0;
  Object.keys(catsOrcadas).forEach(function (cat) {
    if (temCat[cat]) totalGastoOrcado += porCat[cat] || 0;
    else linhas.filter(function (l) { return l.categoria === cat; }).forEach(function (l) { totalGastoOrcado += l.gasto; });
  });
  const semOrcamento = Object.keys(porCat).filter(function (cat) { return !catsOrcadas[cat]; })
    .map(function (cat) { return { categoria: cat, gasto: porCat[cat] }; }).sort(function (a, b) { return b.gasto - a.gasto; });
  return { competencia: comp, linhas: linhas, totalPlanejado: totalPlanejado, totalGastoOrcado: totalGastoOrcado, totalMes: totalMes, semOrcamento: semOrcamento };
}

function acaoOrcamento_(d) {
  const c = ctx_();
  const comp = competenciaValida_(d.competencia);
  const ant = addComp_(comp, -1);
  return Object.assign(orcamentoMes_(c.familia.id, comp), {
    anteriorTemOrcamento: ativos_('orcamentos', c.familia.id).some(function (o) { return o.competencia === ant; })
  });
}

/** Copia o orçamento de um mês para outro (não sobrescreve o que já existe no destino). */
function acaoCopiarOrcamento_(d) {
  const c = ctx_();
  exigir_(c, 'criar');
  const de = competenciaValida_(d.de), para = competenciaValida_(d.para);
  if (de === para) throw erro_('VALIDACAO', 'Escolha meses diferentes.');
  return comLock_(function () {
    const t = tabela_('orcamentos'), fid = c.familia.id;
    const destino = {};
    t.rows.forEach(function (o) { if (o.familia_id === fid && o.excluido !== '1' && o.competencia === para) destino[o.categoria + '||' + (o.subcategoria || '')] = 1; });
    const novos = t.rows.filter(function (o) { return o.familia_id === fid && o.excluido !== '1' && o.competencia === de && !destino[o.categoria + '||' + (o.subcategoria || '')]; })
      .map(function (o) { return marcarCriacao_({ id: uuid_(), familia_id: fid, competencia: para, categoria: o.categoria, subcategoria: o.subcategoria || '', valor_planejado: o.valor_planejado }, c.email); });
    if (!novos.length) throw erro_('VALIDACAO', 'Não há o que copiar: o mês de origem está vazio ou todas as categorias já existem no destino.');
    inserirVarios_(t, novos);
    auditar_(c, 'copiar_orcamento', 'orcamentos', '', { de: de, para: para, linhas: novos.length });
    marcarAlteracao_();
    return { copiadas: novos.length };
  });
}

// ---------------- Metas ----------------

function infoMeta_(m, hoje) {
  const desejado = cent_(m.valor_desejado);
  let acumulado = cent_(m.valor_inicial);
  movs_(m).forEach(function (x) { acumulado += (x.tipo === 'retirada' ? -1 : 1) * cent_(x.valor); });
  const restante = Math.max(0, desejado - acumulado);
  const dias = m.prazo ? diasEntre_(hoje, m.prazo) : null;
  const meses = dias == null ? null : Math.max(1, Math.ceil(dias / 30.4375));
  return {
    desejado: desejado, acumulado: acumulado, restante: restante,
    pct: desejado > 0 ? Math.min(999, Math.round(acumulado * 1000 / desejado) / 10) : 0,
    dias: dias, meses: meses,
    mensal: restante > 0 && dias != null && dias >= 0 ? Math.ceil(restante / meses) : 0,
    atrasada: restante > 0 && dias != null && dias < 0
  };
}

function acaoMovimentarMeta_(d) {
  const c = ctx_();
  exigir_(c, 'criar');
  const tipo = opcao_(d.tipo, ['contribuicao', 'retirada', 'ajuste'], null, 'Tipo');
  const data = dataISO_(d.data, 'Data', true);
  const obs = texto_(d.obs, 'Observação', 200, false);
  return comLock_(function () {
    const t = tabela_('metas');
    const m = t.rows.find(function (x) { return x.id === d.id && x.familia_id === c.familia.id && x.excluido !== '1'; });
    if (!m) throw erro_('NAO_ENCONTRADO', 'Meta não encontrada.');
    if (!podeAlterar_(c, m)) throw erro_('SEM_PERMISSAO', 'Você não tem permissão para alterar esta meta.');
    const atual = infoMeta_(m, hoje_()).acumulado;
    let mov;
    if (tipo === 'ajuste') {
      // No ajuste a pessoa informa o novo total guardado; registramos a diferença.
      const novoTotal = centavos_(d.valor, 'Novo valor guardado', { zero: true });
      const dif = novoTotal - atual;
      if (!dif) throw erro_('VALIDACAO', 'O valor informado é igual ao que já está guardado.');
      mov = { tipo: dif > 0 ? 'ajuste' : 'retirada', valor: paraTexto_(Math.abs(dif)), ajuste: '1' };
    } else {
      const v = centavos_(d.valor, 'Valor');
      if (tipo === 'retirada' && v > atual) throw erro_('VALIDACAO', 'A retirada é maior que o valor guardado (' + brl_(atual) + ').');
      mov = { tipo: tipo, valor: paraTexto_(v) };
    }
    Object.assign(mov, { id: uuid_(), data: data, obs: obs, por: c.email, em: agora_() });
    const lista = movs_(m); lista.push(mov);
    m.movimentos = JSON.stringify(lista);
    const antes = m.status;
    calcular_('metas', m);
    Object.assign(m, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(m) });
    atualizar_(t, m);
    auditar_(c, 'movimentar_meta', 'metas', m.id, { tipo: mov.tipo, valor: mov.valor });
    marcarAlteracao_();
    return { meta: publico_(m), alcancada: antes !== 'concluida' && m.status === 'concluida' };
  });
}

function acaoRemoverMovMeta_(d) {
  const c = ctx_();
  exigir_(c, 'criar');
  return comLock_(function () {
    const t = tabela_('metas');
    const m = t.rows.find(function (x) { return x.id === d.id && x.familia_id === c.familia.id && x.excluido !== '1'; });
    if (!m) throw erro_('NAO_ENCONTRADO', 'Meta não encontrada.');
    if (!podeAlterar_(c, m)) throw erro_('SEM_PERMISSAO', 'Você não tem permissão para alterar esta meta.');
    const lista = movs_(m);
    const mov = lista.find(function (x) { return x.id === d.mov_id; });
    if (!mov) throw erro_('NAO_ENCONTRADO', 'Movimentação não encontrada.');
    const resto = lista.filter(function (x) { return x !== mov; });
    const teste = Object.assign({}, m, { movimentos: JSON.stringify(resto) });
    if (infoMeta_(teste, hoje_()).acumulado < 0) throw erro_('VALIDACAO', 'Remover esta contribuição deixaria a meta com valor negativo. Remova antes a retirada correspondente.');
    m.movimentos = resto.length ? JSON.stringify(resto) : '';
    calcular_('metas', m);
    Object.assign(m, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(m) });
    atualizar_(t, m);
    auditar_(c, 'remover_mov_meta', 'metas', m.id, { tipo: mov.tipo, valor: mov.valor });
    marcarAlteracao_();
    return { meta: publico_(m) };
  });
}

// ---------------- Dívidas ----------------

/** Parcela pelo sistema Price (prestação fixa). */
function parcelaPrice_(S, i, n) {
  if (n <= 0) return S;
  if (!i) return Math.ceil(S / n);
  return Math.round(S * i / (1 - Math.pow(1 + i, -n)));
}
/** Quantas parcelas de valor P quitam o saldo S com juros i. */
function prazoPrice_(S, i, P) {
  if (S <= 0) return 0;
  if (P <= 0) return 999;
  if (!i) return Math.ceil(S / P);
  const x = 1 - i * S / P;
  if (x <= 0) return 999; // a parcela não cobre os juros
  return Math.ceil(-Math.log(x) / Math.log(1 + i) - 1e-9);
}

/**
 * Recalcula a dívida a partir dos dados do contrato e do histórico de pagamentos.
 * Em cada parcela: juros do mês = saldo × taxa; o resto da parcela amortiza o saldo.
 * Amortização extra: abate direto do saldo e recalcula o prazo ou o valor da parcela.
 */
function calcDivida_(r) {
  const i = (parseFloat(r.taxa_juros) || 0) / 100;
  let S = cent_(r.saldo_inicial);
  let P = cent_(r.valor_parcela_inicial);
  let pagas = parseInt(r.parcelas_pagas_inicial, 10) || 0;
  let restantes = Math.max(0, (parseInt(r.qtd_parcelas, 10) || 0) - pagas);
  if (S > 0 && restantes === 0) restantes = prazoPrice_(S, i, P);
  let totalPago = 0, totalJuros = 0;
  const hist = pags_(r).slice().sort(function (a, b) { return (a.data + (a.em || '')).localeCompare(b.data + (b.em || '')); }).map(function (p) {
    const v = cent_(p.valor);
    let juros = 0, amort = v;
    if (p.tipo === 'parcela') {
      juros = Math.min(v, Math.round(S * i));
      amort = v - juros;
      pagas++;
      restantes = Math.max(0, restantes - 1);
    }
    amort = Math.min(amort, S);
    S -= amort;
    totalPago += v; totalJuros += juros;
    if (p.tipo === 'amortizacao' && S > 0) {
      if (p.modo === 'parcela') P = parcelaPrice_(S, i, Math.max(1, restantes));
      else restantes = prazoPrice_(S, i, P);
    }
    if (S <= 0) { S = 0; restantes = 0; }
    return Object.assign({}, p, { juros: juros, amortizacao: amort, saldo_apos: S, numero: p.tipo === 'parcela' ? pagas : null });
  });
  if (S > 0 && restantes > 0) {
    const ajustado = prazoPrice_(S, i, P);
    if (ajustado < restantes && p_ultimaEhAmortizacao_(hist)) restantes = ajustado;
  }
  return { saldo: S, parcela: S > 0 && S < P ? Math.round(S * (1 + i)) : P, pagas: pagas, restantes: restantes, totalPago: totalPago, totalJuros: totalJuros, hist: hist, taxa: i };
}
function p_ultimaEhAmortizacao_(hist) { return hist.length && hist[hist.length - 1].tipo === 'amortizacao'; }

/** Datas das próximas parcelas e projeção do saldo até a quitação. */
function infoDivida_(r, hoje) {
  const x = calcDivida_(r);
  const dia = parseInt(r.dia_vencimento, 10) || 10;
  const dc = r.data_contratacao || hoje;
  // 1ª parcela: no mês seguinte à contratação, no dia de vencimento.
  const p1 = addMes_(+dc.slice(0, 4), +dc.slice(5, 7), 1);
  const venc = function (n) { const p = addMes_(p1.y, p1.m, n - 1); return iso_(p.y, p.m, Math.min(dia, ultimoDia_(p.y, p.m))); };
  const proximas = [];
  let S = x.saldo;
  const projecao = [];
  for (let k = 1; k <= Math.min(x.restantes, 360) && S > 0; k++) {
    const juros = Math.round(S * x.taxa);
    const valor = Math.min(x.parcela, S + juros);
    S = Math.max(0, S + juros - valor);
    const dt = venc(x.pagas + k);
    if (k <= 6) proximas.push({ numero: x.pagas + k, vencimento: dt, valor: valor, atrasada: dt < hoje });
    projecao.push({ data: dt, saldo: S });
  }
  const original = cent_(r.valor_original);
  return {
    saldo: x.saldo, parcela: x.parcela, pagas: x.pagas, restantes: x.restantes, total: x.pagas + x.restantes,
    totalPago: x.totalPago, totalJuros: x.totalJuros, taxa: x.taxa,
    pctQuitado: original > 0 ? Math.max(0, Math.min(100, Math.round((original - x.saldo) * 1000 / original) / 10)) : 0,
    proximas: proximas, historico: x.hist.slice().reverse(),
    evolucao: [{ data: r.data_contratacao, saldo: cent_(r.saldo_inicial) }].concat(x.hist.map(function (h) { return { data: h.data, saldo: h.saldo_apos }; })),
    projecao: projecao.length > 120 ? projecao.filter(function (_, j) { return j % Math.ceil(projecao.length / 120) === 0 || j === projecao.length - 1; }) : projecao,
    quitacaoPrevista: projecao.length ? projecao[projecao.length - 1].data : ''
  };
}

function acaoPagarDivida_(d) {
  const c = ctx_();
  exigir_(c, 'criar');
  const tipo = opcao_(d.tipo, ['parcela', 'amortizacao'], null, 'Tipo');
  const valor = centavos_(d.valor, 'Valor');
  const data = dataISO_(d.data, 'Data', true);
  const conta = contaValida_(d.conta_id, c);
  const modo = opcao_(d.modo, ['prazo', 'parcela'], 'prazo', 'Recálculo');
  return comLock_(function () {
    const t = tabela_('dividas');
    const r = t.rows.find(function (x) { return x.id === d.id && x.familia_id === c.familia.id && x.excluido !== '1'; });
    if (!r) throw erro_('NAO_ENCONTRADO', 'Dívida não encontrada.');
    if (!podeAlterar_(c, r)) throw erro_('SEM_PERMISSAO', 'Você não tem permissão para alterar esta dívida.');
    if (r.status === 'cancelada') throw erro_('VALIDACAO', 'Esta dívida está cancelada.');
    const x = calcDivida_(r);
    if (x.saldo <= 0) throw erro_('VALIDACAO', 'Esta dívida já está quitada.');
    const maximo = tipo === 'parcela' ? x.saldo + Math.round(x.saldo * x.taxa) : x.saldo;
    if (valor > maximo) throw erro_('VALIDACAO', 'O valor é maior que o necessário para quitar (' + brl_(maximo) + ').');
    const pag = { id: uuid_(), tipo: tipo, valor: paraTexto_(valor), data: data, conta_id: conta, modo: tipo === 'amortizacao' ? modo : '', por: c.email, em: agora_(), despesa_id: '' };
    if (d.lancarDespesa) {
      const desp = marcarCriacao_({
        id: uuid_(), familia_id: c.familia.id,
        descricao: (r.nome + (tipo === 'parcela' ? ' — parcela ' + (x.pagas + 1) + '/' + (x.pagas + x.restantes) : ' — amortização extra')).slice(0, 130),
        valor: pag.valor, categoria: 'Financeiro', subcategoria: r.tipo === 'financiamento' ? 'Financiamentos' : 'Empréstimos',
        responsavel: c.usuario.nome, forma_pagamento: '', conta_id: conta, data_compra: data, data_vencimento: '', data_pagamento: data,
        status: 'paga', recorrencia: 'nenhuma', parcela: '', observacoes: 'Lançado pela tela de Dívidas', anexo_url: '', demo: r.demo || ''
      }, c.email);
      inserirVarios_(tabela_('despesas'), [desp]);
      pag.despesa_id = desp.id;
    }
    const lista = pags_(r); lista.push(pag);
    r.pagamentos = JSON.stringify(lista);
    calcular_('dividas', r);
    Object.assign(r, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(r) });
    atualizar_(t, r);
    auditar_(c, tipo === 'parcela' ? 'pagar_parcela' : 'amortizar', 'dividas', r.id, { valor: pag.valor, data: data, modo: pag.modo });
    marcarAlteracao_();
    return { divida: publico_(r), info: infoDivida_(r, hoje_()) };
  });
}

function acaoRemoverPagDivida_(d) {
  const c = ctx_();
  exigir_(c, 'criar');
  return comLock_(function () {
    const t = tabela_('dividas');
    const r = t.rows.find(function (x) { return x.id === d.id && x.familia_id === c.familia.id && x.excluido !== '1'; });
    if (!r) throw erro_('NAO_ENCONTRADO', 'Dívida não encontrada.');
    if (!podeAlterar_(c, r)) throw erro_('SEM_PERMISSAO', 'Você não tem permissão para alterar esta dívida.');
    const lista = pags_(r);
    const p = lista.find(function (x) { return x.id === d.pagamento_id; });
    if (!p) throw erro_('NAO_ENCONTRADO', 'Pagamento não encontrado.');
    const resto = lista.filter(function (x) { return x !== p; });
    r.pagamentos = resto.length ? JSON.stringify(resto) : '';
    calcular_('dividas', r);
    Object.assign(r, { atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(r) });
    atualizar_(t, r);
    if (p.despesa_id) {
      const td = tabela_('despesas');
      const desp = td.rows.find(function (x) { return x.id === p.despesa_id && x.familia_id === c.familia.id; });
      if (desp && desp.excluido !== '1') {
        Object.assign(desp, { excluido: '1', atualizado_por: c.email, atualizado_em: agora_(), versao: proximaVersao_(desp) });
        atualizar_(td, desp);
      }
    }
    auditar_(c, 'remover_pagamento', 'dividas', r.id, { valor: p.valor, data: p.data });
    marcarAlteracao_();
    return { divida: publico_(r) };
  });
}

/** Tudo o que a tela de Metas/Dívidas precisa, com os cálculos prontos. */
function acaoPlanejamento_() {
  const c = ctx_();
  const fid = c.familia.id, hoje = hoje_();
  const ordemPrior = { alta: 0, media: 1, baixa: 2 }, ordemSt = { ativa: 0, pausada: 1, concluida: 2, cancelada: 3 };
  const metas = ativos_('metas', fid).map(function (m) {
    return Object.assign(publico_(m), { info: infoMeta_(m, hoje), historico: movs_(m).slice().reverse().sort(function (a, b) { return String(b.data).localeCompare(String(a.data)); }) });
  }).sort(function (a, b) {
    return (ordemSt[a.status] - ordemSt[b.status]) || (ordemPrior[a.prioridade] - ordemPrior[b.prioridade]) || String(a.prazo || '9999').localeCompare(String(b.prazo || '9999'));
  });
  const dividas = ativos_('dividas', fid).map(function (r) { return Object.assign(publico_(r), { info: infoDivida_(r, hoje) }); })
    .sort(function (a, b) { return (a.status === 'ativa' ? 0 : 1) - (b.status === 'ativa' ? 0 : 1) || b.info.saldo - a.info.saldo; });
  const ativas = dividas.filter(function (x) { return x.status === 'ativa'; });
  return {
    hoje: hoje, metas: metas, dividas: dividas,
    resumoMetas: {
      desejado: metas.filter(function (m) { return m.status !== 'cancelada'; }).reduce(function (s, m) { return s + m.info.desejado; }, 0),
      acumulado: metas.filter(function (m) { return m.status !== 'cancelada'; }).reduce(function (s, m) { return s + m.info.acumulado; }, 0),
      mensal: metas.filter(function (m) { return m.status === 'ativa'; }).reduce(function (s, m) { return s + m.info.mensal; }, 0)
    },
    resumoDividas: {
      saldo: ativas.reduce(function (s, x) { return s + x.info.saldo; }, 0),
      original: ativas.reduce(function (s, x) { return s + cent_(x.valor_original); }, 0),
      pago: dividas.reduce(function (s, x) { return s + x.info.totalPago; }, 0),
      juros: dividas.reduce(function (s, x) { return s + x.info.totalJuros; }, 0),
      parcelasMes: ativas.reduce(function (s, x) { return s + (x.info.proximas[0] ? x.info.proximas[0].valor : 0); }, 0),
      proximas: ativas.map(function (x) { return x.info.proximas[0] ? Object.assign({ divida: x.nome, id: x.id }, x.info.proximas[0]) : null; })
        .filter(Boolean).sort(function (a, b) { return a.vencimento.localeCompare(b.vencimento); })
    }
  };
}

// ============================================================
// 7c. RELATÓRIOS
// ============================================================

const RELATORIOS = {
  rxd: 'Receitas × despesas', receitas_mes: 'Receitas por mês', despesas_mes: 'Despesas por mês',
  categorias: 'Despesas por categoria', pessoas: 'Receitas e despesas por pessoa', cartoes: 'Gastos por cartão',
  faturas: 'Histórico de faturas', saldo: 'Evolução do saldo', economias: 'Evolução das economias',
  orcamento: 'Orçamento planejado × realizado', dividas: 'Dívidas e parcelas restantes', metas: 'Evolução das metas'
};
const COMPARAVEIS = ['rxd', 'receitas_mes', 'despesas_mes', 'categorias', 'pessoas', 'cartoes', 'economias', 'orcamento'];

function acaoRelatorio_(d) {
  const c = ctx_();
  return relatorio_(c, d);
}

function lerPeriodoRel_(d, pre) {
  const ini = dataISO_(d[pre + 'inicio'], 'Início', true), fim = dataISO_(d[pre + 'fim'], 'Fim', true);
  if (ini > fim) throw erro_('VALIDACAO', 'A data inicial deve ser anterior à final.');
  if (diasEntre_(ini, fim) > 3700) throw erro_('VALIDACAO', 'Período muito longo (máximo 10 anos).');
  return { ini: ini, fim: fim };
}

function mesesDe_(ini, fim) {
  const out = [];
  let p = { y: +ini.slice(0, 4), m: +ini.slice(5, 7) };
  const yf = +fim.slice(0, 4), mf = +fim.slice(5, 7);
  while ((p.y * 12 + p.m) <= (yf * 12 + mf) && out.length < 121) {
    out.push({ comp: p.y + '-' + String(p.m).padStart(2, '0'), a: iso_(p.y, p.m, 1), b: iso_(p.y, p.m, ultimoDia_(p.y, p.m)) });
    p = addMes_(p.y, p.m, 1);
  }
  out[0].a = ini > out[0].a ? ini : out[0].a;
  out[out.length - 1].b = fim < out[out.length - 1].b ? fim : out[out.length - 1].b;
  return out;
}

function relatorio_(c, d) {
  const tipo = d.tipo;
  if (!RELATORIOS[tipo]) throw erro_('VALIDACAO', 'Relatório inválido.');
  const A = lerPeriodoRel_(d, '');
  const B = (d.comparar && COMPARAVEIS.indexOf(tipo) >= 0) ? lerPeriodoRel_(d, 'b_') : null;
  const f = d.filtros || {};
  const fid = c.familia.id, hoje = hoje_();
  const passa = function (r) {
    return (!f.responsavel || r.responsavel === f.responsavel) && (!f.categoria || r.categoria === f.categoria) && (!f.conta_id || r.conta_id === f.conta_id);
  };
  const rec = ativos_('receitas', fid).filter(function (r) { return r.status !== 'cancelada' && passa(r); });
  const des = ativos_('despesas', fid).filter(function (r) { return r.status !== 'cancelada' && passa(r); });
  const V = function (r) { return cent_(r.valor); };
  const recEm = function (a, b) { return rec.filter(function (r) { return r.data >= a && r.data <= b; }); };
  const desEm = function (a, b) { return des.filter(function (r) { return r.data_compra >= a && r.data_compra <= b; }); };
  const soma = function (arr) { return arr.reduce(function (s, r) { return s + V(r); }, 0); };
  const somaQ = function (arr, ent) { return arr.reduce(function (s, r) { return s + quitado_(r, ent); }, 0); };
  const rotMes = function (comp) { return mesAno_(comp); };
  const rotP = function (P) { return dataBR_(P.ini) + ' a ' + dataBR_(P.fim); };
  const out = { tipo: tipo, titulo: RELATORIOS[tipo], periodo: rotP(A), comparacao: B ? rotP(B) : '', colunas: [], linhas: [], totais: null, grafico: null, nota: '' };
  const col = function (id, rotulo, tp) { return { id: id, rotulo: rotulo, tipo: tp || 'moeda' }; };
  const variacao = function (a, b) { return b ? Math.round((a - b) * 1000 / Math.abs(b)) / 10 : null; };

  // Relatórios mês a mês. Na comparação, o 1º mês de A é comparado com o 1º mês de B, e assim por diante.
  const mensal = function (fn, colsDef, graf) {
    const ma = mesesDe_(A.ini, A.fim), mb = B ? mesesDe_(B.ini, B.fim) : [];
    out.colunas = [col('mes', 'Mês', 'texto')].concat(colsDef);
    if (B) out.colunas = out.colunas.concat([col('mes_b', 'Mês comparado', 'texto'), col(colsDef[0].id + '_b', colsDef[0].rotulo + ' (comparação)'), col('var', 'Variação', 'pct')]);
    out.linhas = ma.map(function (m, i) {
      const l = Object.assign({ mes: rotMes(m.comp) }, fn(m.a, m.b, m.comp));
      if (B) {
        const mb_ = mb[i];
        if (mb_) { const lb = fn(mb_.a, mb_.b, mb_.comp); l.mes_b = rotMes(mb_.comp); l[colsDef[0].id + '_b'] = lb[colsDef[0].id]; l['var'] = variacao(l[colsDef[0].id], lb[colsDef[0].id]); }
      }
      return l;
    });
    out.totais = { mes: 'Total' };
    colsDef.forEach(function (cd) { if (cd.tipo === 'moeda' && !cd.semTotal) out.totais[cd.id] = out.linhas.reduce(function (s, l) { return s + (l[cd.id] || 0); }, 0); });
    if (B) { const k = colsDef[0].id + '_b'; out.totais[k] = out.linhas.reduce(function (s, l) { return s + (l[k] || 0); }, 0); out.totais['var'] = variacao(out.totais[colsDef[0].id], out.totais[k]); }
    out.grafico = { tipo: graf.tipo || 'bar', labels: out.linhas.map(function (l) { return l.mes; }), series: graf.series.map(function (s) { return { nome: s[1], dados: out.linhas.map(function (l) { return l[s[0]] == null ? null : l[s[0]]; }), cor: s[2], tipo: s[3] }; }) };
    if (B) out.grafico.series.push({ nome: colsDef[0].rotulo + ' (comparação)', dados: out.linhas.map(function (l) { return l[colsDef[0].id + '_b'] == null ? null : l[colsDef[0].id + '_b']; }), cor: 'cinza' });
  };

  // Ranking por grupo (categoria, pessoa, cartão), com comparação lado a lado.
  const ranking = function (fnMapa, rotuloGrupo, rotuloValor) {
    const ma = fnMapa(A.ini, A.fim), mb = B ? fnMapa(B.ini, B.fim) : {};
    const nomes = Object.keys(Object.assign({}, ma, mb));
    const total = nomes.reduce(function (s, k) { return s + (ma[k] || 0); }, 0);
    out.colunas = [col('nome', rotuloGrupo, 'texto'), col('valor', rotuloValor), col('pct', '% do total', 'pct')];
    if (B) out.colunas = out.colunas.concat([col('valor_b', rotuloValor + ' (comparação)'), col('dif', 'Diferença'), col('var', 'Variação', 'pct')]);
    out.linhas = nomes.map(function (k) {
      const l = { nome: k, valor: ma[k] || 0, pct: total ? Math.round((ma[k] || 0) * 1000 / total) / 10 : 0 };
      if (B) { l.valor_b = mb[k] || 0; l.dif = l.valor - l.valor_b; l['var'] = variacao(l.valor, l.valor_b); }
      return l;
    }).sort(function (a, b) { return b.valor - a.valor || (b.valor_b || 0) - (a.valor_b || 0); });
    out.totais = { nome: 'Total', valor: total, pct: total ? 100 : 0 };
    if (B) { out.totais.valor_b = out.linhas.reduce(function (s, l) { return s + l.valor_b; }, 0); out.totais.dif = total - out.totais.valor_b; out.totais['var'] = variacao(total, out.totais.valor_b); }
    out.grafico = B
      ? { tipo: 'barh', labels: out.linhas.slice(0, 12).map(function (l) { return l.nome; }), series: [{ nome: 'Período', dados: out.linhas.slice(0, 12).map(function (l) { return l.valor; }), cor: 'azul' }, { nome: 'Comparação', dados: out.linhas.slice(0, 12).map(function (l) { return l.valor_b; }), cor: 'cinza' }] }
      : { tipo: 'doughnut', labels: out.linhas.slice(0, 10).map(function (l) { return l.nome; }), series: [{ nome: rotuloValor, dados: out.linhas.slice(0, 10).map(function (l) { return l.valor; }) }] };
  };
  const agrupar = function (arr, chave) { const m = {}; arr.forEach(function (r) { const k = chave(r) || '(sem informação)'; m[k] = (m[k] || 0) + V(r); }); return m; };

  if (tipo === 'rxd') {
    mensal(function (a, b) { const r = recEm(a, b), x = desEm(a, b); return { receitas: soma(r), despesas: soma(x), resultado: soma(r) - soma(x) }; },
      [col('receitas', 'Receitas'), col('despesas', 'Despesas'), col('resultado', 'Resultado')],
      { series: [['receitas', 'Receitas', 'verde'], ['despesas', 'Despesas', 'vermelho'], ['resultado', 'Resultado', 'azul', 'line']] });
  } else if (tipo === 'receitas_mes') {
    mensal(function (a, b) { const r = recEm(a, b); const q = somaQ(r, 'receitas'); return { total: soma(r), recebido: q, previsto: soma(r) - q }; },
      [col('total', 'Total'), col('recebido', 'Recebido'), col('previsto', 'A receber')], { series: [['recebido', 'Recebido', 'verde'], ['previsto', 'A receber', 'verdeClaro']] });
    out.grafico.empilhado = !B;
  } else if (tipo === 'despesas_mes') {
    mensal(function (a, b) { const x = desEm(a, b); const q = somaQ(x, 'despesas'); const cart = soma(x.filter(function (r) { return r.cartao_id; })); return { total: soma(x), pago: q, aberto: soma(x) - q, cartao: cart }; },
      [col('total', 'Total'), col('pago', 'Pago'), col('aberto', 'Em aberto'), col('cartao', 'Das quais no cartão')], { series: [['pago', 'Pago', 'vermelho'], ['aberto', 'Em aberto', 'laranja']] });
    out.grafico.empilhado = !B;
  } else if (tipo === 'economias') {
    let acc = 0, accB = 0;
    mensal(function (a, b) { const e = somaQ(recEm(a, b), 'receitas') - soma(desEm(a, b)); return { economia: e }; }, [col('economia', 'Economia do mês'), col('acumulada', 'Acumulada', 'moeda')],
      { series: [['economia', 'Economia do mês', 'verde'], ['acumulada', 'Acumulada', 'azul', 'line']] });
    out.linhas.forEach(function (l) { acc += l.economia; l.acumulada = acc; });
    out.grafico.series[1].dados = out.linhas.map(function (l) { return l.acumulada; });
    delete out.totais.acumulada;
    out.nota = 'Economia = receitas recebidas − despesas do mês (pagas ou não).';
    void accB;
  } else if (tipo === 'categorias') {
    ranking(function (a, b) { return agrupar(desEm(a, b), function (r) { return r.categoria; }); }, 'Categoria', 'Gasto');
  } else if (tipo === 'pessoas') {
    ranking(function (a, b) { return agrupar(desEm(a, b), function (r) { return r.responsavel; }); }, 'Pessoa', 'Despesas');
    const recP = agrupar(recEm(A.ini, A.fim), function (r) { return r.responsavel; });
    if (!B) {
      Object.keys(recP).forEach(function (k) { if (!out.linhas.some(function (l) { return l.nome === k; })) out.linhas.push({ nome: k, valor: 0, pct: 0 }); });
      out.colunas.push(col('receitas', 'Receitas'), col('saldo', 'Receitas − despesas'));
      out.linhas.forEach(function (l) { l.receitas = recP[l.nome] || 0; l.saldo = l.receitas - l.valor; });
      out.totais.receitas = out.linhas.reduce(function (s, l) { return s + l.receitas; }, 0);
      out.totais.saldo = out.totais.receitas - out.totais.valor;
      out.grafico = { tipo: 'barh', labels: out.linhas.map(function (l) { return l.nome; }), series: [{ nome: 'Receitas', dados: out.linhas.map(function (l) { return l.receitas; }), cor: 'verde' }, { nome: 'Despesas', dados: out.linhas.map(function (l) { return l.valor; }), cor: 'vermelho' }] };
    }
  } else if (tipo === 'cartoes') {
    const nomes = {};
    tabela_('cartoes').rows.forEach(function (k) { if (k.familia_id === fid) nomes[k.id] = k.nome; });
    ranking(function (a, b) { return agrupar(desEm(a, b).filter(function (r) { return r.cartao_id; }), function (r) { return nomes[r.cartao_id]; }); }, 'Cartão', 'Gasto no período');
    out.nota = 'Cada parcela conta no mês da sua compra/parcela.';
  } else if (tipo === 'faturas') {
    const fs = faturasFamilia_(fid).filter(function (x) { return x.vencimento >= A.ini && x.vencimento <= A.fim; });
    out.colunas = [col('cartao', 'Cartão', 'texto'), col('fatura', 'Fatura', 'texto'), col('vencimento', 'Vencimento', 'data'), col('total', 'Total'), col('pago', 'Pago'), col('restante', 'Restante'), col('situacao', 'Situação', 'texto')];
    const ST = { aberta: 'Aberta', fechada: 'Fechada', vencida: 'Vencida', paga: 'Paga' };
    out.linhas = fs.map(function (x) { return { cartao: x.cartao_nome, fatura: mesAno_(x.competencia), vencimento: x.vencimento, total: x.total, pago: x.pago, restante: Math.max(0, x.restante), situacao: ST[x.status] || x.status }; });
    out.totais = { cartao: 'Total', total: out.linhas.reduce(function (s, l) { return s + l.total; }, 0), pago: out.linhas.reduce(function (s, l) { return s + l.pago; }, 0), restante: out.linhas.reduce(function (s, l) { return s + l.restante; }, 0) };
    const meses = mesesDe_(A.ini, A.fim);
    out.grafico = { tipo: 'bar', empilhado: true, labels: meses.map(function (m) { return mesAno_(m.comp); }), series: [{ nome: 'Pago', cor: 'verde', dados: meses.map(function (m) { return fs.filter(function (x) { return x.competencia === m.comp; }).reduce(function (s, x) { return s + Math.min(x.pago, x.total); }, 0); }) }, { nome: 'Em aberto', cor: 'laranja', dados: meses.map(function (m) { return fs.filter(function (x) { return x.competencia === m.comp; }).reduce(function (s, x) { return s + Math.max(0, x.restante); }, 0); }) }] };
  } else if (tipo === 'saldo') {
    const contas = ativos_('contas_bancarias', fid);
    const movs = movimentos_(fid);
    const base = {}; contas.forEach(function (k) { base[k.id] = k.data_saldo_inicial || '0000-01-01'; });
    const menor = contas.map(function (k) { return base[k.id]; }).sort()[0] || '0000-01-01';
    const saldoEm = function (ref) {
      let s = 0;
      contas.forEach(function (k) { if (base[k.id] <= ref) s += cent_(k.saldo_inicial); });
      movs.forEach(function (m) { if (m.data <= ref && m.data >= (base[m.conta_id] || menor)) s += m.valor; });
      return s;
    };
    const meses = mesesDe_(A.ini, A.fim);
    out.colunas = [col('mes', 'Mês', 'texto'), col('entradas', 'Entradas'), col('saidas', 'Saídas'), col('saldo', 'Saldo no fim do mês')];
    out.linhas = meses.filter(function (m) { return m.a <= hoje; }).map(function (m) {
      const ref = m.b < hoje ? m.b : hoje;
      const per = movs.filter(function (x) { return x.tipo !== 'transf' && x.data >= m.a && x.data <= ref; });
      return { mes: mesAno_(m.comp), entradas: per.reduce(function (s, x) { return s + (x.valor > 0 ? x.valor : 0); }, 0), saidas: per.reduce(function (s, x) { return s + (x.valor < 0 ? -x.valor : 0); }, 0), saldo: saldoEm(ref) };
    });
    out.totais = { mes: 'Total', entradas: out.linhas.reduce(function (s, l) { return s + l.entradas; }, 0), saidas: out.linhas.reduce(function (s, l) { return s + l.saidas; }, 0) };
    out.grafico = { tipo: 'line', labels: out.linhas.map(function (l) { return l.mes; }), series: [{ nome: 'Saldo', dados: out.linhas.map(function (l) { return l.saldo; }), cor: 'azul' }] };
    out.nota = 'Saldo de todas as contas da família somadas. Meses futuros não aparecem. Filtros de pessoa e categoria não se aplicam a este relatório.';
  } else if (tipo === 'orcamento') {
    const porCat = function (a, b) {
      const meses = mesesDe_(a, b); const m = {};
      meses.forEach(function (mm) {
        const o = orcamentoMes_(fid, mm.comp);
        o.linhas.forEach(function (l) { if (l.subcategoria) return; const k = l.categoria; m[k] = m[k] || { plan: 0, gasto: 0 }; m[k].plan += l.planejado; m[k].gasto += l.gasto; });
      });
      return m;
    };
    const ma = porCat(A.ini, A.fim), mb = B ? porCat(B.ini, B.fim) : {};
    out.colunas = [col('nome', 'Categoria', 'texto'), col('planejado', 'Planejado'), col('realizado', 'Realizado'), col('diferenca', 'Diferença (sobra/estouro)'), col('pct', '% usado', 'pct')];
    if (B) out.colunas.push(col('realizado_b', 'Realizado (comparação)'), col('var', 'Variação', 'pct'));
    out.linhas = Object.keys(Object.assign({}, ma, mb)).map(function (k) {
      const x = ma[k] || { plan: 0, gasto: 0 };
      const l = { nome: k, planejado: x.plan, realizado: x.gasto, diferenca: x.plan - x.gasto, pct: x.plan ? Math.round(x.gasto * 1000 / x.plan) / 10 : null };
      if (B) { l.realizado_b = (mb[k] || { gasto: 0 }).gasto; l['var'] = variacao(l.realizado, l.realizado_b); }
      return l;
    }).sort(function (a, b) { return b.planejado - a.planejado; });
    const tp = out.linhas.reduce(function (s, l) { return s + l.planejado; }, 0), tr = out.linhas.reduce(function (s, l) { return s + l.realizado; }, 0);
    out.totais = { nome: 'Total', planejado: tp, realizado: tr, diferenca: tp - tr, pct: tp ? Math.round(tr * 1000 / tp) / 10 : null };
    if (B) { out.totais.realizado_b = out.linhas.reduce(function (s, l) { return s + l.realizado_b; }, 0); out.totais['var'] = variacao(tr, out.totais.realizado_b); }
    out.grafico = { tipo: 'barh', labels: out.linhas.map(function (l) { return l.nome; }), series: [{ nome: 'Planejado', dados: out.linhas.map(function (l) { return l.planejado; }), cor: 'cinza' }, { nome: 'Realizado', dados: out.linhas.map(function (l) { return l.realizado; }), cor: 'azul' }] };
    out.nota = 'Considera os orçamentos por categoria cadastrados em cada mês do período. Os filtros de pessoa/conta não se aplicam.';
  } else if (tipo === 'dividas') {
    const ds = ativos_('dividas', fid).filter(function (x) { return x.status !== 'cancelada'; }).map(function (r) { return { r: r, i: infoDivida_(r, hoje) }; });
    out.colunas = [col('nome', 'Dívida', 'texto'), col('credor', 'Credor', 'texto'), col('original', 'Valor original'), col('pago', 'Total pago'), col('saldo', 'Saldo devedor'), col('parcela', 'Parcela atual'), col('parcelas', 'Pagas / total', 'texto'), col('proxima', 'Próxima parcela', 'data'), col('quitacao', 'Quitação prevista', 'data')];
    out.linhas = ds.map(function (x) {
      return { nome: x.r.nome, credor: x.r.credor, original: cent_(x.r.valor_original), pago: x.i.totalPago, saldo: x.i.saldo, parcela: x.i.saldo > 0 ? x.i.parcela : 0, parcelas: x.i.pagas + ' / ' + x.i.total, proxima: x.i.proximas[0] ? x.i.proximas[0].vencimento : '', quitacao: x.i.quitacaoPrevista };
    }).sort(function (a, b) { return b.saldo - a.saldo; });
    out.totais = { nome: 'Total', original: out.linhas.reduce(function (s, l) { return s + l.original; }, 0), pago: out.linhas.reduce(function (s, l) { return s + l.pago; }, 0), saldo: out.linhas.reduce(function (s, l) { return s + l.saldo; }, 0), parcela: out.linhas.reduce(function (s, l) { return s + l.parcela; }, 0) };
    out.grafico = { tipo: 'barh', empilhado: true, labels: out.linhas.map(function (l) { return l.nome; }), series: [{ nome: 'Já quitado', dados: out.linhas.map(function (l) { return Math.max(0, l.original - l.saldo); }), cor: 'verde' }, { nome: 'Saldo devedor', dados: out.linhas.map(function (l) { return l.saldo; }), cor: 'vermelho' }] };
    out.nota = 'Posição atual das dívidas (o período não se aplica).';
  } else if (tipo === 'metas') {
    const ms = ativos_('metas', fid).filter(function (m) { return m.status !== 'cancelada'; });
    const ST = { ativa: 'Em andamento', concluida: 'Alcançada', pausada: 'Pausada' };
    out.colunas = [col('nome', 'Meta', 'texto'), col('desejado', 'Valor desejado'), col('acumulado', 'Guardado'), col('pct', '% concluído', 'pct'), col('periodo', 'Guardado no período'), col('prazo', 'Prazo', 'data'), col('mensal', 'Necessário por mês'), col('situacao', 'Situação', 'texto')];
    out.linhas = ms.map(function (m) {
      const i = infoMeta_(m, hoje);
      const noPer = movs_(m).filter(function (x) { return x.data >= A.ini && x.data <= A.fim; }).reduce(function (s, x) { return s + (x.tipo === 'retirada' ? -1 : 1) * cent_(x.valor); }, 0);
      return { nome: m.nome, desejado: i.desejado, acumulado: i.acumulado, pct: i.pct, periodo: noPer, prazo: m.prazo, mensal: i.mensal, situacao: i.atrasada ? 'Prazo vencido' : (ST[m.status] || m.status) };
    });
    out.totais = { nome: 'Total', desejado: out.linhas.reduce(function (s, l) { return s + l.desejado; }, 0), acumulado: out.linhas.reduce(function (s, l) { return s + l.acumulado; }, 0), periodo: out.linhas.reduce(function (s, l) { return s + l.periodo; }, 0), mensal: out.linhas.reduce(function (s, l) { return s + l.mensal; }, 0) };
    out.totais.pct = out.totais.desejado ? Math.round(out.totais.acumulado * 1000 / out.totais.desejado) / 10 : 0;
    // Evolução mensal do total guardado em todas as metas.
    const meses = mesesDe_(A.ini, A.fim);
    out.grafico = { tipo: 'line', labels: meses.map(function (m) { return mesAno_(m.comp); }), series: [{ nome: 'Total guardado nas metas', cor: 'verde', dados: meses.map(function (mm) {
      if (mm.a > hoje) return null;
      return ms.reduce(function (s, m) { return s + (m.data_inicio <= mm.b ? cent_(m.valor_inicial) : 0) + movs_(m).filter(function (x) { return x.data <= mm.b; }).reduce(function (t, x) { return t + (x.tipo === 'retirada' ? -1 : 1) * cent_(x.valor); }, 0); }, 0);
    }) }] };
  }
  out.filtros = f;
  return out;
}

/** Cria uma planilha no Google Drive de quem pediu, com o relatório, e devolve os links de download. */
function acaoExportarRelatorio_(d) {
  const c = ctx_();
  const r = relatorio_(c, d);
  const nome = 'Relatório ' + r.titulo + ' · ' + r.periodo.replace(/\//g, '-') + ' · ' + c.familia.nome;
  const ss = SpreadsheetApp.create(nome.slice(0, 150));
  const sh = ss.getSheets()[0];
  sh.setName('Relatório');
  const cols = r.colunas;
  const fmt = function (v, tp) {
    if (v == null || v === '') return '';
    if (tp === 'moeda') return v / 100;
    if (tp === 'pct') return v / 100;
    if (tp === 'data') return dataBR_(v);
    return v;
  };
  const cab = [[r.titulo], ['Família: ' + c.familia.nome + ' · Período: ' + r.periodo + (r.comparacao ? ' · Comparação: ' + r.comparacao : '')], ['Gerado em ' + dataBR_(hoje_()) + ' por ' + c.usuario.nome + (r.nota ? ' · ' + r.nota : '')], ['']];
  const largura = Math.max(1, cols.length);
  sh.getRange(1, 1, cab.length, largura).setValues(cab.map(function (l) { const x = l.slice(); while (x.length < largura) x.push(''); return x; }));
  sh.getRange(1, 1).setFontWeight('bold').setFontSize(14);
  const ini = cab.length + 1;
  const linhas = [cols.map(function (cc) { return cc.rotulo; })].concat(r.linhas.map(function (l) { return cols.map(function (cc) { return fmt(l[cc.id], cc.tipo); }); }));
  if (r.totais) linhas.push(cols.map(function (cc) { return fmt(r.totais[cc.id], cc.tipo); }));
  sh.getRange(ini, 1, linhas.length, largura).setValues(linhas);
  sh.getRange(ini, 1, 1, largura).setFontWeight('bold').setBackground('#1e293b').setFontColor('#ffffff');
  if (r.totais) sh.getRange(ini + linhas.length - 1, 1, 1, largura).setFontWeight('bold').setBackground('#eef2f8');
  cols.forEach(function (cc, j) {
    if (linhas.length < 2) return;
    if (cc.tipo === 'moeda') sh.getRange(ini + 1, j + 1, linhas.length - 1, 1).setNumberFormat('"R$" #,##0.00;-"R$" #,##0.00');
    if (cc.tipo === 'pct') sh.getRange(ini + 1, j + 1, linhas.length - 1, 1).setNumberFormat('0.0%');
  });
  sh.setFrozenRows(ini);
  try { sh.autoResizeColumns(1, largura); } catch (e) { /* opcional */ }
  SpreadsheetApp.flush();
  const id = ss.getId(), gid = sh.getSheetId();
  const base = 'https://docs.google.com/spreadsheets/d/' + id + '/export?';
  auditar_(c, 'exportar_relatorio', 'relatorios', id, { tipo: r.tipo, periodo: r.periodo });
  return {
    url: ss.getUrl(), nome: nome,
    xlsx: base + 'format=xlsx',
    csv: base + 'format=csv&gid=' + gid,
    pdf: base + 'format=pdf&gid=' + gid + '&portrait=false&size=A4&fitw=true&gridlines=false&sheetnames=false&printtitle=false'
  };
}

function dataBR_(iso) { return iso ? iso.slice(8, 10) + '/' + iso.slice(5, 7) + '/' + iso.slice(0, 4) : ''; }

// ============================================================
// 7d. NOTIFICAÇÕES, HISTÓRICO, BACKUP, AUTOMAÇÃO E SEGURANÇA (Etapa 4)
//     Nenhuma destas funções exige permissões novas do Google.
// ============================================================

const TIPOS_NOTIF = {
  vencimentos: 'Contas a pagar vencendo ou atrasadas',
  recebimentos: 'Receitas previstas ainda não recebidas',
  faturas: 'Faturas de cartão a pagar',
  cartoes: 'Limite do cartão (70%, 85% e 95%)',
  orcamento: 'Orçamento perto do limite ou estourado',
  metas: 'Metas alcançadas e prazos próximos',
  dividas: 'Parcelas de dívidas a vencer',
  duplicados: 'Possíveis lançamentos duplicados'
};

/** Preferências e notificações lidas ficam nas propriedades de cada usuário (funciona até para quem só visualiza). */
function propsUsuario_() { return PropertiesService.getUserProperties(); }
function lerJSONProp_(props, chave, padrao) {
  try { const v = props.getProperty(chave); return v ? JSON.parse(v) : padrao; } catch (e) { return padrao; }
}
function prefsNotif_(c) {
  const salvas = lerJSONProp_(propsUsuario_(), 'notif_prefs:' + c.familia.id, {});
  const out = {};
  Object.keys(TIPOS_NOTIF).forEach(function (k) { out[k] = salvas[k] !== false; });
  return out;
}
function lidasNotif_(c) { return lerJSONProp_(propsUsuario_(), 'notif_lidas:' + c.familia.id, []); }

/** Gera as notificações de hoje a partir dos dados. Cada uma tem uma chave estável, então "lida" não volta a aparecer. */
function gerarNotificacoes_(c) {
  const fid = c.familia.id, hoje = hoje_();
  const amanha = addDias_(hoje, 1), d3 = addDias_(hoje, 3), d7 = addDias_(hoje, 7);
  const out = [];
  const faixaVenc = function (v) {
    if (v < hoje) return ['atrasada', 'perigo', 'venceu em ' + dataBR_(v) + ' (' + plural_(diasEntre_(v, hoje), 'dia', 'dias') + ' de atraso)'];
    if (v === hoje) return ['hoje', 'perigo', 'vence hoje'];
    if (v === amanha) return ['amanha', 'alerta', 'vence amanhã'];
    if (v <= d3) return ['3dias', 'alerta', 'vence em ' + dataBR_(v)];
    if (v <= d7) return ['7dias', 'info', 'vence em ' + dataBR_(v)];
    return null;
  };
  const des = ativos_('despesas', fid), rec = ativos_('receitas', fid);
  des.forEach(function (r) {
    if (r.cartao_id || r.status === 'cancelada') return;
    const rest = restante_(r, 'despesas'); if (rest <= 0) return;
    const v = r.data_vencimento || r.data_compra;
    const f = faixaVenc(v); if (!f) return;
    out.push({ chave: 'venc:' + r.id + ':' + v + ':' + f[0], tipo: 'vencimentos', nivel: f[1], data: v, titulo: r.descricao + ' ' + f[2], texto: brl_(rest) + (pags_(r).length ? ' restantes' : '') + ' · ' + r.responsavel, acao: { rota: 'pagar' } });
  });
  rec.forEach(function (r) {
    if (r.status === 'cancelada') return;
    const rest = restante_(r, 'receitas');
    if (rest <= 0 || r.data >= hoje) return;
    out.push({ chave: 'rec:' + r.id + ':' + r.data, tipo: 'recebimentos', nivel: 'info', data: r.data, titulo: r.descricao + ': previsto para ' + dataBR_(r.data) + ' e ainda não recebido', texto: brl_(rest) + ' · ' + r.responsavel, acao: { rota: 'pagar', aba: 'receber' } });
  });
  const fats = faturasFamilia_(fid);
  fats.forEach(function (f) {
    if (f.restante <= 0 || f.status === 'aberta') return;
    const x = faixaVenc(f.vencimento); if (!x) return;
    out.push({ chave: 'fat:' + f.cartao_id + ':' + f.competencia + ':' + x[0], tipo: 'faturas', nivel: x[1], data: f.vencimento, titulo: 'Fatura ' + f.cartao_nome + ' (' + mesAno_(f.competencia) + ') ' + x[2], texto: brl_(f.restante) + ' a pagar', acao: { rota: 'cartoes', cartao_id: f.cartao_id, comp: f.competencia } });
  });
  ativos_('cartoes', fid).forEach(function (k) {
    const limite = cent_(k.limite);
    if (k.status === 'inativo' || limite <= 0) return;
    const usado = fats.filter(function (f) { return f.cartao_id === k.id; }).reduce(function (s, f) { return s + Math.max(0, f.restante); }, 0);
    const pct = Math.round(usado * 100 / limite);
    const faixa = pct >= 95 ? 95 : pct >= 85 ? 85 : pct >= 70 ? 70 : 0;
    if (faixa) out.push({ chave: 'lim:' + k.id + ':' + faixa + ':' + hoje.slice(0, 7), tipo: 'cartoes', nivel: faixa >= 95 ? 'perigo' : 'alerta', data: hoje, titulo: 'Cartão ' + k.nome + ' com ' + pct + '% do limite usado', texto: brl_(usado) + ' de ' + brl_(limite) + ' · disponível ' + brl_(limite - usado), acao: { rota: 'cartoes', cartao_id: k.id } });
  });
  const orc = orcamentoMes_(fid, hoje.slice(0, 7));
  orc.linhas.forEach(function (l) {
    if (l.nivel < 70) return;
    const nome = l.subcategoria ? l.categoria + ' ▸ ' + l.subcategoria : l.categoria;
    const t = l.nivel > 100 ? 'estourado' : l.nivel === 100 ? 'atingido' : 'com ' + fmtPct_(l.pct) + ' usado';
    out.push({ chave: 'orc:' + l.id + ':' + l.nivel, tipo: 'orcamento', nivel: l.nivel >= 100 ? 'perigo' : 'alerta', data: hoje, titulo: 'Orçamento de ' + nome + ' ' + t, texto: brl_(l.gasto) + ' de ' + brl_(l.planejado) + ' em ' + mesAno_(hoje.slice(0, 7)), acao: { rota: 'metas', aba: 'orcamento' } });
  });
  ativos_('metas', fid).forEach(function (m) {
    const i = infoMeta_(m, hoje);
    if (m.status === 'concluida') out.push({ chave: 'meta_ok:' + m.id, tipo: 'metas', nivel: 'ok', data: (m.atualizado_em || '').slice(0, 10), titulo: '🎉 Meta alcançada: ' + m.nome, texto: brl_(i.acumulado) + ' guardados', acao: { rota: 'metas', aba: 'metas' } });
    else if (m.status === 'ativa' && m.prazo && i.restante > 0) {
      if (i.dias < 0) out.push({ chave: 'meta_venc:' + m.id + ':' + m.prazo, tipo: 'metas', nivel: 'alerta', data: m.prazo, titulo: 'O prazo da meta ' + m.nome + ' terminou', texto: 'Faltaram ' + brl_(i.restante) + '. Que tal ajustar o prazo?', acao: { rota: 'metas', aba: 'metas' } });
      else if (i.dias <= 30) out.push({ chave: 'meta_prazo:' + m.id + ':' + m.prazo, tipo: 'metas', nivel: 'info', data: m.prazo, titulo: 'Meta ' + m.nome + ': prazo em ' + plural_(i.dias, 'dia', 'dias'), texto: 'Faltam ' + brl_(i.restante), acao: { rota: 'metas', aba: 'metas' } });
    }
  });
  ativos_('dividas', fid).forEach(function (d) {
    if (d.status !== 'ativa') return;
    const px = infoDivida_(d, hoje).proximas[0]; if (!px) return;
    const f = faixaVenc(px.vencimento); if (!f) return;
    out.push({ chave: 'div:' + d.id + ':' + px.numero + ':' + f[0], tipo: 'dividas', nivel: f[1], data: px.vencimento, titulo: d.nome + ': parcela ' + px.numero + ' ' + f[2], texto: brl_(px.valor), acao: { rota: 'metas', aba: 'dividas' } });
  });
  duplicados_(des, rec, hoje).forEach(function (p) { out.push(p); });
  return out;
}

/**
 * Possíveis duplicados: mesmo tipo, mesmo valor, descrição parecida (ou mesma categoria)
 * e datas com até 3 dias de diferença. Ignora parcelas, recorrências e compras do mesmo grupo.
 */
function duplicados_(des, rec, hoje) {
  const limite = addDias_(hoje, -60);
  const norm = function (s) { return String(s || '').toLowerCase().normalize('NFD').replace(/[̀-ͯ]/g, '').replace(/\[demo\]/g, '').replace(/\(\d+\/\d+\)/g, '').replace(/[^a-z0-9]+/g, ' ').trim(); };
  const out = [];
  [['despesas', des, 'data_compra'], ['receitas', rec, 'data']].forEach(function (x) {
    const lista = x[1].filter(function (r) { return r.status !== 'cancelada' && r[x[2]] >= limite && !r.serie_id && !r.compra_id && !r.grupo_id; });
    const porValor = {};
    lista.forEach(function (r) { (porValor[r.valor] = porValor[r.valor] || []).push(r); });
    Object.keys(porValor).forEach(function (v) {
      const g = porValor[v].sort(function (a, b) { return a[x[2]].localeCompare(b[x[2]]); });
      for (let i = 0; i < g.length; i++) {
        for (let j = i + 1; j < g.length; j++) {
          const a = g[i], b = g[j];
          if (diasEntre_(a[x[2]], b[x[2]]) > 3) break;
          const parecida = norm(a.descricao) === norm(b.descricao) || (a.categoria === b.categoria && (a.subcategoria || '') === (b.subcategoria || '') && norm(a.descricao).slice(0, 5) === norm(b.descricao).slice(0, 5));
          if (!parecida) continue;
          const ids = [a.id, b.id].sort();
          out.push({
            chave: 'dup:' + ids.join(':'), tipo: 'duplicados', nivel: 'alerta', data: b[x[2]],
            titulo: 'Possível lançamento duplicado: ' + a.descricao, texto: (x[0] === 'despesas' ? 'Duas despesas' : 'Duas receitas') + ' de ' + brl_(cent_(v)) + ' em ' + dataBR_(a[x[2]]) + (a[x[2]] !== b[x[2]] ? ' e ' + dataBR_(b[x[2]]) : '') + ', lançadas por ' + nomeDe_(a.criado_por, a.familia_id) + (a.criado_por !== b.criado_por ? ' e ' + nomeDe_(b.criado_por, b.familia_id) : ''),
            acao: { duplicado: { entidade: x[0], registros: [publico_(a), publico_(b)] } }
          });
        }
      }
    });
  });
  return out;
}

function acaoNotificacoes_() {
  const c = ctx_();
  const prefs = prefsNotif_(c);
  const lidas = {};
  lidasNotif_(c).forEach(function (k) { lidas[k] = 1; });
  const PESO = { perigo: 0, alerta: 1, info: 2, ok: 3 };
  const itens = gerarNotificacoes_(c).filter(function (n) { return prefs[n.tipo]; })
    .map(function (n) { return Object.assign(n, { lida: !!lidas[n.chave] }); })
    .sort(function (a, b) { return (a.lida ? 1 : 0) - (b.lida ? 1 : 0) || PESO[a.nivel] - PESO[b.nivel] || String(a.data).localeCompare(String(b.data)); });
  return { itens: itens, naoLidas: itens.filter(function (n) { return !n.lida; }).length, prefs: prefs, tipos: TIPOS_NOTIF };
}

function acaoMarcarNotificacoes_(d) {
  const c = ctx_();
  const atuais = gerarNotificacoes_(c).map(function (n) { return n.chave; });
  const existe = {};
  atuais.forEach(function (k) { existe[k] = 1; });
  // Guarda só as chaves que ainda existem (a lista não cresce para sempre).
  let lidas = lidasNotif_(c).filter(function (k) { return existe[k]; });
  const alvo = d.todas ? atuais : (Array.isArray(d.chaves) ? d.chaves.map(String) : []);
  if (d.desmarcar) lidas = lidas.filter(function (k) { return alvo.indexOf(k) < 0; });
  else alvo.forEach(function (k) { if (existe[k] && lidas.indexOf(k) < 0) lidas.push(k); });
  propsUsuario_().setProperty('notif_lidas:' + c.familia.id, JSON.stringify(lidas.slice(-400)));
  return acaoNotificacoes_();
}

function acaoPrefsNotificacoes_(d) {
  const c = ctx_();
  const p = {};
  Object.keys(TIPOS_NOTIF).forEach(function (k) { p[k] = !(d.prefs && d.prefs[k] === false); });
  propsUsuario_().setProperty('notif_prefs:' + c.familia.id, JSON.stringify(p));
  return acaoNotificacoes_();
}

// ---------------- Histórico de alterações ----------------

function acaoHistorico_(d) {
  const c = ctx_();
  exigir_(c, 'administrar');
  const pagina = Math.max(1, parseInt(d.pagina, 10) || 1), tam = 50;
  const ini = d.inicio ? dataISO_(d.inicio, 'Início', false) : '', fim = d.fim ? dataISO_(d.fim, 'Fim', false) : '';
  const nomes = {};
  tabela_('usuarios').rows.forEach(function (u) { if (u.familia_id === c.familia.id) nomes[u.email] = u.nome; });
  let rows = tabela_('historico').rows.filter(function (h) {
    if (h.familia_id !== c.familia.id && !(h.acao === 'edicao_manual' && !h.familia_id)) return false;
    if (d.entidade && h.entidade !== d.entidade) return false;
    if (d.usuario && h.usuario !== d.usuario) return false;
    if (d.acao && h.acao !== d.acao) return false;
    const dia = String(h.data_hora).slice(0, 10);
    if (ini && dia < ini) return false;
    if (fim && dia > fim) return false;
    return true;
  });
  rows.sort(function (a, b) { return String(b.data_hora).localeCompare(String(a.data_hora)) || b._linha - a._linha; });
  const total = rows.length;
  const itens = rows.slice((pagina - 1) * tam, pagina * tam).map(function (h) {
    let det = {};
    try { det = JSON.parse(h.detalhes || '{}'); } catch (e) { det = {}; }
    return { data_hora: h.data_hora, usuario: h.usuario, nome: nomes[h.usuario] || (h.usuario === 'automacao@sistema' ? 'Automação' : h.usuario), acao: h.acao, entidade: h.entidade, registro_id: h.registro_id, detalhes: det };
  });
  const usuarios = {};
  tabela_('historico').rows.forEach(function (h) { if (h.familia_id === c.familia.id) usuarios[h.usuario] = nomes[h.usuario] || h.usuario; });
  return { itens: itens, total: total, pagina: pagina, paginas: Math.max(1, Math.ceil(total / tam)), usuarios: usuarios };
}

// ---------------- Cópia de segurança e exportação ----------------

function lerConfig_(fid, chave) {
  const r = tabela_('configuracoes').rows.find(function (x) { return x.familia_id === fid && x.chave === chave && x.excluido !== '1'; });
  return r ? r.valor : '';
}

/** Copia todas as abas para uma planilha nova no Drive de quem pediu. */
function fazerBackup_(c) {
  const src = ss_();
  const agora = agora_();
  const nome = 'Cópia de segurança · ' + c.familia.nome + ' · ' + dataBR_(agora.slice(0, 10)) + ' ' + agora.slice(11, 16);
  const dest = SpreadsheetApp.create(nome);
  const inicial = dest.getSheets()[0];
  const abas = [];
  Object.keys(ENTIDADES).forEach(function (ent) {
    const sh = src.getSheetByName(ENTIDADES[ent].aba);
    if (!sh) return;
    const cp = sh.copyTo(dest);
    cp.setName(ENTIDADES[ent].aba);
    try { if (cp.isSheetHidden()) cp.showSheet(); } catch (e) { /* opcional */ }
    abas.push({ nome: ENTIDADES[ent].aba, gid: cp.getSheetId() });
  });
  dest.deleteSheet(inicial);
  SpreadsheetApp.flush();
  const id = dest.getId(), base = 'https://docs.google.com/spreadsheets/d/' + id + '/export?';
  const info = { data: agora, url: dest.getUrl(), id: id, por: c.email, nome: nome };
  let lista = [];
  try { lista = JSON.parse(lerConfig_(c.familia.id, 'backups') || '[]'); } catch (e) { lista = []; }
  lista.unshift(info);
  setConfig_(c, 'backups', JSON.stringify(lista.slice(0, 20)));
  auditar_(c, 'backup', 'sistema', id, { nome: nome, abas: abas.length });
  return Object.assign({}, info, {
    xlsx: base + 'format=xlsx',
    abas: abas.map(function (a) { return { nome: a.nome, csv: base + 'format=csv&gid=' + a.gid }; })
  });
}

function acaoBackup_() {
  const c = ctx_();
  exigir_(c, 'administrar');
  return comLock_(function () { return fazerBackup_(c); });
}

// ---------------- Automação diária ----------------

/**
 * Executada por um acionador diário (criado pelo administrador no editor do Apps Script).
 * Gera os lançamentos recorrentes e, se ativado, faz a cópia de segurança semanal.
 */
function automacaoDiaria() {
  const fam = familia_();
  if (!fam) return;
  const c = { email: 'automacao@sistema', usuario: { id: '', nome: 'Automação', email: 'automacao@sistema' }, familia: fam, perfil: 'admin' };
  const reg = { data_hora: agora_(), recorrencias: 0, backup: '', erro: '' };
  const lock = LockService.getScriptLock();
  if (!lock.tryLock(30000)) { reg.erro = 'Outra operação estava em andamento.'; return reg; }
  try {
    limparCacheTabelas_();
    reg.recorrencias = gerarRecorrencias_(c, null);
    PropertiesService.getScriptProperties().setProperty('RECORRENCIA_DIA', hoje_());
    if (lerConfig_(fam.id, 'backup_semanal') === '1') {
      let ultimo = '';
      try { ultimo = (JSON.parse(lerConfig_(fam.id, 'backups') || '[]')[0] || {}).data || ''; } catch (e) { ultimo = ''; }
      if (!ultimo || diasEntre_(ultimo.slice(0, 10), hoje_()) >= 7) reg.backup = fazerBackup_(c).url;
    }
  } catch (e) {
    reg.erro = String(e && e.message || e).slice(0, 200);
  }
  try {
    let log = [];
    try { log = JSON.parse(lerConfig_(fam.id, 'automacao_log') || '[]'); } catch (e) { log = []; }
    log.unshift(reg);
    setConfig_(c, 'automacao_log', JSON.stringify(log.slice(0, 30)));
    auditar_(c, 'automacao', 'sistema', '', reg);
    if (reg.recorrencias || reg.backup) marcarAlteracao_();
    SpreadsheetApp.flush();
  } finally {
    lock.releaseLock();
  }
  return reg;
}

function acaoConfigSistema_(d) {
  const c = ctx_();
  exigir_(c, 'administrar');
  return comLock_(function () {
    if (d.backup_semanal !== undefined) setConfig_(c, 'backup_semanal', d.backup_semanal ? '1' : '0');
    auditar_(c, 'configurar', 'sistema', '', { backup_semanal: !!d.backup_semanal });
    return acaoSegurancaInterna_(c);
  });
}

function acaoRodarAutomacao_() {
  const c = ctx_();
  exigir_(c, 'administrar');
  const r = automacaoDiaria();
  limparCacheTabelas_();
  return Object.assign({ execucao: r }, acaoSegurancaInterna_(c));
}

// ---------------- Segurança e acesso ----------------

const DESCR_PROTECAO = 'Finanças da Família — use o aplicativo para alterar os dados';

function acaoSeguranca_() {
  const c = ctx_();
  exigir_(c, 'administrar');
  return acaoSegurancaInterna_(c);
}

function acaoSegurancaInterna_(c) {
  const ss = ss_(), fid = c.familia.id, hoje = hoje_();
  const emails = function (lista) { return (lista || []).map(function (u) { return String(u.getEmail ? u.getEmail() : u).toLowerCase(); }).filter(Boolean); };
  let dono = '';
  try { dono = String(ss.getOwner().getEmail()).toLowerCase(); } catch (e) { dono = ''; }
  const editores = emails(ss.getEditors()).filter(function (e) { return e !== dono; });
  const leitores = emails(ss.getViewers()).filter(function (e) { return e !== dono && editores.indexOf(e) < 0; });
  const membros = tabela_('usuarios').rows.filter(function (u) { return u.familia_id === fid && u.excluido !== '1'; });
  const porEmail = {};
  membros.forEach(function (u) { porEmail[u.email.toLowerCase()] = u; });
  const problemas = [];
  const acesso = [];
  editores.concat(leitores).forEach(function (e) {
    const u = porEmail[e], editor = editores.indexOf(e) >= 0;
    acesso.push({ email: e, nome: u ? u.nome : '', papel: editor ? 'editor' : 'leitor', perfil: u ? u.perfil : '', status: u ? u.status : 'desconhecido' });
    if (!u) problemas.push({ nivel: 'perigo', codigo: 'estranho', email: e, texto: e + ' tem acesso à planilha, mas não faz parte da família no app.', correcao: 'remover', rotulo: 'Remover acesso' });
    else if (u.status === 'revogado') problemas.push({ nivel: 'perigo', codigo: 'revogado', email: e, texto: u.nome + ' teve o acesso revogado no app, mas ainda pode abrir a planilha.', correcao: 'remover', rotulo: 'Remover acesso' });
    else if (u.perfil === 'visualizador' && editor) problemas.push({ nivel: 'alerta', codigo: 'leitor_editor', email: e, texto: u.nome + ' é Visualizador no app, mas pode editar a planilha.', correcao: 'leitor', rotulo: 'Deixar só leitura' });
  });
  membros.forEach(function (u) {
    const e = u.email.toLowerCase();
    if (e === dono || u.status !== 'ativo') return;
    if (editores.indexOf(e) < 0 && leitores.indexOf(e) < 0) problemas.push({ nivel: 'alerta', codigo: 'sem_acesso', email: e, texto: u.nome + ' está ativo no app, mas a planilha não está compartilhada com ' + e + ' (o app não vai abrir para essa pessoa).', correcao: 'compartilhar', rotulo: 'Compartilhar' });
    else if (u.perfil !== 'visualizador' && editores.indexOf(e) < 0) problemas.push({ nivel: 'alerta', codigo: 'sem_edicao', email: e, texto: u.nome + ' é ' + (u.perfil === 'admin' ? 'Administrador' : 'Membro') + ' no app, mas só tem leitura na planilha (não vai conseguir salvar).', correcao: 'compartilhar', rotulo: 'Dar acesso de edição' });
  });
  const protegida = lerConfig_(fid, 'protegida') === '1';
  if (!protegida) problemas.push({ nivel: 'alerta', codigo: 'sem_protecao', texto: 'As abas de dados não estão protegidas: quem tem acesso pode editar a planilha à mão sem nenhum aviso.', correcao: 'proteger', rotulo: 'Proteger a planilha' });
  let backups = [];
  try { backups = JSON.parse(lerConfig_(fid, 'backups') || '[]'); } catch (e) { backups = []; }
  if (!backups.length || diasEntre_(backups[0].data.slice(0, 10), hoje) > 30) problemas.push({ nivel: 'info', codigo: 'sem_backup', texto: backups.length ? 'A última cópia de segurança tem mais de 30 dias.' : 'Ainda não foi feita nenhuma cópia de segurança.', correcao: 'backup', rotulo: 'Fazer cópia agora' });
  let log = [];
  try { log = JSON.parse(lerConfig_(fid, 'automacao_log') || '[]'); } catch (e) { log = []; }
  const manuais = tabela_('historico').rows.filter(function (h) { return h.acao === 'edicao_manual' && String(h.data_hora).slice(0, 10) >= addDias_(hoje, -30); }).length;
  if (manuais) problemas.push({ nivel: 'alerta', codigo: 'edicao_manual', texto: plural_(manuais, 'edição feita', 'edições feitas') + ' direto na planilha nos últimos 30 dias. Confira no histórico de alterações.', correcao: 'historico', rotulo: 'Ver histórico' });
  return {
    dono: dono, acesso: acesso, problemas: problemas, protegida: protegida,
    backups: backups.slice(0, 10), backupSemanal: lerConfig_(fid, 'backup_semanal') === '1',
    automacao: { log: log.slice(0, 10), ultima: log[0] || null, ativaRecente: !!(log[0] && diasEntre_(log[0].data_hora.slice(0, 10), hoje) <= 2) },
    urlPlanilha: ss.getUrl()
  };
}

function acaoCorrigirAcesso_(d) {
  const c = ctx_();
  exigir_(c, 'administrar');
  const email = String(d.email || '').trim().toLowerCase();
  const ss = ss_();
  let dono = '';
  try { dono = String(ss.getOwner().getEmail()).toLowerCase(); } catch (e) { dono = ''; }
  if (!email || email === dono || email === c.email) throw erro_('VALIDACAO', 'Não é possível alterar o acesso do dono da planilha.');
  return comLock_(function () {
    const u = usuarioPorEmail_(email, c.familia.id);
    let aviso = '';
    if (d.acao === 'remover') descompartilhar_(email);
    else if (d.acao === 'leitor') { try { ss.removeEditor(email); } catch (e) { /* ignora */ } ss.addViewer(email); }
    else if (d.acao === 'compartilhar') {
      if (!u || u.status !== 'ativo') throw erro_('VALIDACAO', 'Essa pessoa não está ativa na família.');
      aviso = compartilhar_(email, u.perfil);
    } else throw erro_('ACAO_INVALIDA', 'Correção inválida.');
    auditar_(c, 'corrigir_acesso', 'usuarios', u ? u.id : '', { email: email, acao: d.acao });
    return Object.assign({ aviso: aviso }, acaoSegurancaInterna_(c));
  });
}

/**
 * Protege as abas de dados com AVISO: quem tentar editar a planilha à mão vê um alerta.
 * O app continua gravando normalmente. As abas de dados ficam ocultas e uma aba "LEIA-ME" explica o motivo.
 */
function acaoProteger_(d) {
  const c = ctx_();
  exigir_(c, 'administrar');
  const ativar = d.ativar !== false;
  return comLock_(function () {
    const ss = ss_();
    let leia = ss.getSheetByName('LEIA-ME');
    if (ativar && !leia) {
      leia = ss.insertSheet('LEIA-ME', 0);
      leia.getRange(1, 1, 6, 1).setValues([
        ['⚠️ Finanças da Família — não edite esta planilha diretamente'],
        ['Os dados ficam em abas ocultas e são gravados pelo aplicativo, que valida, evita duplicidades e registra quem alterou cada coisa.'],
        ['Edições feitas à mão podem quebrar cálculos e ficam registradas no histórico de alterações.'],
        ['Para usar: ' + (urlApp_() || 'abra o link do aplicativo')],
        ['Administrador: ' + c.usuario.nome + ' (' + c.email + ')'],
        ['Para desfazer a proteção: Configurações ▸ Segurança, no aplicativo.']
      ]);
      leia.getRange(1, 1).setFontWeight('bold').setFontSize(14);
    }
    let n = 0;
    Object.keys(ENTIDADES).forEach(function (ent) {
      const sh = ss.getSheetByName(ENTIDADES[ent].aba);
      if (!sh) return;
      const minhas = sh.getProtections(SpreadsheetApp.ProtectionType.SHEET).filter(function (p) { return p.getDescription() === DESCR_PROTECAO; });
      if (ativar) {
        if (!minhas.length) { sh.protect().setDescription(DESCR_PROTECAO).setWarningOnly(true); n++; }
        if (!sh.isSheetHidden()) sh.hideSheet();
      } else {
        minhas.forEach(function (p) { p.remove(); n++; });
        if (sh.isSheetHidden()) sh.showSheet();
      }
    });
    if (!ativar && leia) ss.deleteSheet(leia);
    setConfig_(c, 'protegida', ativar ? '1' : '0');
    auditar_(c, ativar ? 'proteger_planilha' : 'desproteger_planilha', 'sistema', '', { abas: n });
    return acaoSegurancaInterna_(c);
  });
}

/** Acionador simples: registra no histórico qualquer edição feita à mão nas abas de dados. */
function onEdit(e) {
  try {
    if (!e || !e.range) return;
    const sh = e.range.getSheet();
    const ent = Object.keys(ENTIDADES).find(function (k) { return ENTIDADES[k].aba === sh.getName(); });
    if (!ent || ent === 'historico') return;
    _cache.ss = e.source || SpreadsheetApp.getActiveSpreadsheet();
    let email = '';
    try { email = e.user && e.user.getEmail ? String(e.user.getEmail()) : ''; } catch (x) { email = ''; }
    const fam = familia_();
    inserirVarios_(tabela_('historico'), [{
      id: uuid_(), familia_id: fam ? fam.id : '', data_hora: agora_(), usuario: email || 'edição direta (usuário não identificado)',
      acao: 'edicao_manual', entidade: ent, registro_id: '',
      detalhes: JSON.stringify({ aba: sh.getName(), celula: e.range.getA1Notation(), linhas: e.range.getNumRows() })
    }]);
    marcarAlteracao_();
  } catch (err) { /* nunca atrapalhar quem está editando */ }
}

// ============================================================
// 8. DADOS DE DEMONSTRAÇÃO (marcados com demo=1 e "[DEMO]")
// ============================================================

function acaoGerarDemo_() {
  const c = ctx_();
  exigir_(c, 'administrar');
  return comLock_(function () {
    const existe = tabela_('receitas').rows.some(function (r) { return r.demo === '1' && r.familia_id === c.familia.id && r.excluido !== '1'; });
    if (existe) throw erro_('JA_EXISTE', 'Os dados de demonstração já estão na planilha.');
    gerarDemo_(c);
    auditar_(c, 'gerar_demo', 'sistema', '', {});
    marcarAlteracao_();
    return pacote_(c);
  });
}

function acaoRemoverDemo_() {
  const c = ctx_();
  exigir_(c, 'administrar');
  return comLock_(function () {
    let total = 0;
    ['receitas', 'despesas', 'contas_bancarias', 'transferencias', 'compras_cartao', 'faturas', 'cartoes', 'orcamentos', 'metas', 'dividas'].forEach(function (ent) {
      const t = tabela_(ent);
      const linhas = t.rows.filter(function (r) { return r.demo === '1' && r.familia_id === c.familia.id; })
        .map(function (r) { return r._linha; }).sort(function (a, b) { return b - a; });
      total += linhas.length;
      // Remove blocos contíguos de baixo para cima (poucas chamadas à planilha).
      let i = 0;
      while (i < linhas.length) {
        let j = i;
        while (j + 1 < linhas.length && linhas[j + 1] === linhas[j] - 1) j++;
        t.sh.deleteRows(linhas[j], j - i + 1);
        i = j + 1;
      }
    });
    auditar_(c, 'remover_demo', 'sistema', '', { registros: total });
    marcarAlteracao_();
    limparCacheTabelas_();
    return Object.assign({ removidos: total }, pacote_(c));
  });
}

function gerarDemo_(c) {
  const email = c.email, fid = c.familia.id, eu = c.usuario.nome;
  const outro = 'Pessoa 2 (demo)';
  const hoje = hoje_();
  const Y = +hoje.slice(0, 4), M = +hoje.slice(5, 7);
  const ini = addMes_(Y, M, -5);
  const extra = function (o) { return marcarCriacao_(Object.assign(o, { id: uuid_(), familia_id: fid, demo: '1' }), email); };

  const contas = [
    { nome: '[DEMO] Conta corrente', instituicao: 'Banco Exemplo', titular: eu, tipo: 'corrente', saldo_inicial: '2500.00', data_saldo_inicial: iso_(ini.y, ini.m, 1), status: 'ativa' },
    { nome: '[DEMO] Carteira', instituicao: '', titular: outro, tipo: 'carteira', saldo_inicial: '150.00', data_saldo_inicial: iso_(ini.y, ini.m, 1), status: 'ativa' }
  ].map(extra);
  inserirVarios_(tabela_('contas_bancarias'), contas);
  const cc = contas[0].id, cart = contas[1].id;

  const rec = [], des = [];
  for (let k = 5; k >= 0; k--) {
    const p = addMes_(Y, M, -k);
    const dia = function (n) { return iso_(p.y, p.m, Math.min(n, ultimoDia_(p.y, p.m))); };
    const statusR = function (dt) { return dt <= hoje ? 'recebida' : 'prevista'; };
    rec.push({ descricao: '[DEMO] Salário', valor: '4200.00', categoria: 'Salário', responsavel: eu, conta_id: cc, data: dia(5), status: statusR(dia(5)), recorrencia: 'mensal' });
    rec.push({ descricao: '[DEMO] Salário', valor: '3100.00', categoria: 'Salário', responsavel: outro, conta_id: cc, data: dia(10), status: statusR(dia(10)), recorrencia: 'mensal' });
    if (k % 2 === 0) rec.push({ descricao: '[DEMO] Vendas por afiliados', valor: paraTexto_(38000 + k * 4500), categoria: 'Renda extra', responsavel: eu, conta_id: cc, data: dia(18), status: statusR(dia(18)), recorrencia: 'nenhuma' });

    const lista = [
      ['Aluguel', 'Moradia', 'Aluguel ou financiamento', 180000, 10, eu, 'boleto', true],
      ['Energia elétrica', 'Moradia', 'Energia elétrica', 16500 + k * 1250, 15, outro, 'pix', true],
      ['Internet', 'Moradia', 'Internet', 11990, 20, eu, 'debito', true],
      ['Plano de saúde', 'Saúde', 'Plano de saúde', 54000, 8, outro, 'boleto', true],
      ['Supermercado', 'Alimentação', 'Supermercado', 68000 + k * 3700, 6, eu, 'debito', false],
      ['Supermercado', 'Alimentação', 'Supermercado', 41000 + k * 2100, 21, outro, 'debito', false],
      ['Combustível', 'Transporte', 'Combustível', 26000 + k * 1500, 12, outro, 'debito', false],
      ['Farmácia', 'Saúde', 'Farmácia', 8900 + k * 700, 14, eu, 'pix', false],
      ['Delivery', 'Alimentação', 'Delivery', 12000 + k * 900, 24, eu, 'pix', false],
      ['Cinema e passeio', 'Pessoal', 'Lazer', 15000 + (k % 3) * 4000, 27, outro, 'dinheiro', false]
    ];
    lista.forEach(function (x) {
      const dt = dia(x[4]);
      const fixo = x[7];
      // Mês atual: energia fica em aberto (e vencida) para demonstrar os alertas.
      const emAberto = dt > hoje || (k === 0 && x[0] === 'Energia elétrica');
      des.push({
        descricao: '[DEMO] ' + x[0], valor: paraTexto_(x[3]), categoria: x[1], subcategoria: x[2],
        responsavel: x[5], forma_pagamento: x[6], conta_id: x[6] === 'dinheiro' ? cart : cc,
        data_compra: dt, data_vencimento: fixo ? dt : '', data_pagamento: emAberto ? '' : dt,
        status: emAberto ? 'pendente' : 'paga', recorrencia: fixo ? 'mensal' : 'nenhuma', parcela: '', observacoes: '', anexo_url: ''
      });
    });
  }
  inserirVarios_(tabela_('receitas'), rec.map(extra));
  inserirVarios_(tabela_('despesas'), des.map(extra));

  // Etapa 2: um cartão com compras parceladas, faturas passadas pagas e uma transferência.
  const cartao = extra({ nome: '[DEMO] Cartão Roxo', instituicao: 'Banco Exemplo', titular: eu, limite: '5000.00', dia_fechamento: '3', dia_vencimento: '10', conta_pagamento_id: cc, cor: '#7c3aed', status: 'ativo' });
  inserirVarios_(tabela_('cartoes'), [cartao]);
  const dt = function (meses, dia) { const p = addMes_(Y, M, meses); const x = iso_(p.y, p.m, Math.min(dia, ultimoDia_(p.y, p.m))); return x > hoje ? hoje : x; };
  [
    ['[DEMO] Geladeira', 240000, 10, 'Moradia', 'Manutenção', dt(-4, 15), eu],
    ['[DEMO] Roupas da escola', 36000, 3, 'Família', 'Roupas', dt(-1, 20), outro],
    ['[DEMO] Streaming', 3990, 1, 'Pessoal', 'Assinaturas', dt(-2, 12), eu],
    ['[DEMO] Streaming', 3990, 1, 'Pessoal', 'Assinaturas', dt(-1, 12), eu],
    ['[DEMO] Streaming', 3990, 1, 'Pessoal', 'Assinaturas', dt(0, 12), eu],
    ['[DEMO] Restaurante', 18750, 1, 'Alimentação', 'Restaurante', dt(0, 2), outro]
  ].forEach(function (x) {
    criarCompra_(c, cartao, { descricao: x[0], total: x[1], parcelas: x[2], categoria: x[3], subcategoria: x[4], data_compra: x[5], responsavel: x[6], observacoes: '' }, uuid_(), true);
  });
  faturasFamilia_(fid).forEach(function (f) {
    if (f.cartao_id !== cartao.id || f.vencimento >= hoje || f.restante <= 0) return;
    const fat = sincronizarFatura_(c, cartao, f.competencia);
    fat.pagamentos = JSON.stringify([{ id: uuid_(), data: f.vencimento, valor: paraTexto_(f.restante), conta_id: cc, por: email, em: agora_() }]);
    atualizar_(tabela_('faturas'), fat);
    sincronizarFatura_(c, cartao, f.competencia);
  });
  const transf = [];
  for (let k = 5; k >= 0; k--) transf.push(extra({ pessoa_origem: eu, pessoa_destino: outro, conta_origem_id: cc, conta_destino_id: cart, valor: '250.00', data: dt(-k, 2), descricao: '[DEMO] Dinheiro para a carteira' }));
  inserirVarios_(tabela_('transferencias'), transf);

  // Etapa 3: orçamento do mês atual e do anterior, uma meta com contribuições e uma dívida com pagamentos.
  const orc = [];
  [0, -1].forEach(function (k) {
    const p = addMes_(Y, M, k), comp = p.y + '-' + String(p.m).padStart(2, '0');
    [['Moradia', '', '3000.00'], ['Alimentação', '', '1500.00'], ['Transporte', '', '400.00'], ['Saúde', '', '650.00'], ['Pessoal', '', '350.00'], ['Alimentação', 'Delivery', '120.00']]
      .forEach(function (x) { orc.push(extra({ competencia: comp, categoria: x[0], subcategoria: x[1], valor_planejado: x[2] })); });
  });
  inserirVarios_(tabela_('orcamentos'), orc);
  const mv = function (meses, valor, tipo) { return { id: uuid_(), tipo: tipo || 'contribuicao', valor: valor, data: dt(meses, 6), obs: '', por: email, em: agora_() }; };
  const metas = [
    extra({ nome: '[DEMO] Reserva de emergência', tipo: 'reserva', valor_desejado: '15000.00', valor_inicial: '4000.00', data_inicio: dt(-5, 1), prazo: iso_(Y + 1, 6, 30), responsavel: eu, prioridade: 'alta', descricao: '6 meses de despesas fixas', status: 'ativa', conta_id: '', movimentos: JSON.stringify([mv(-4, '500.00'), mv(-3, '500.00'), mv(-2, '800.00'), mv(-1, '500.00', 'retirada'), mv(0, '700.00')]) }),
    extra({ nome: '[DEMO] Viagem de férias', tipo: 'viagem', valor_desejado: '6000.00', valor_inicial: '1200.00', data_inicio: dt(-3, 1), prazo: addDias_(hoje, 25), responsavel: outro, prioridade: 'media', descricao: '', status: 'ativa', conta_id: '', movimentos: JSON.stringify([mv(-2, '600.00'), mv(-1, '600.00')]) })
  ];
  metas.forEach(function (m) { calcular_('metas', m); });
  inserirVarios_(tabela_('metas'), metas);
  const pc = addMes_(Y, M, -14);
  const divida = extra({
    nome: '[DEMO] Financiamento do carro', credor: 'Banco Exemplo', tipo: 'financiamento', valor_original: '30000.00', saldo_inicial: '24500.00', taxa_juros: '1.49',
    qtd_parcelas: '48', valor_parcela_inicial: '882.00', parcelas_pagas_inicial: '8', data_contratacao: iso_(pc.y, pc.m, 20), dia_vencimento: '20', conta_id: cc, status: 'ativa',
    pagamentos: JSON.stringify([5, 4, 3, 2, 1].map(function (k) { return { id: uuid_(), tipo: 'parcela', valor: '882.00', data: dt(-k, 20), conta_id: cc, modo: '', por: email, em: agora_(), despesa_id: '' }; })
      .concat([{ id: uuid_(), tipo: 'amortizacao', valor: '2000.00', data: dt(-2, 21), conta_id: cc, modo: 'prazo', por: email, em: agora_(), despesa_id: '' }]))
  });
  calcular_('dividas', divida);
  inserirVarios_(tabela_('dividas'), [divida]);
}

function categoriasPadrao_(fid, email) {
  const lista = [];
  ['Salário', 'Adiantamento salarial', 'Horas extras', 'Décimo terceiro', 'Férias', 'Renda extra', 'Vendas', 'Comissões',
    'Trabalhos autônomos', 'Benefícios', 'Rendimentos de investimentos', 'Reembolsos', 'Outros']
    .forEach(function (n) { lista.push({ tipo: 'receita', grupo: '', nome: n, cor: '#16a34a', icone: '💰' }); });
  const grupos = [
    ['Moradia', '#2563eb', '🏠', ['Aluguel ou financiamento', 'Condomínio', 'Água', 'Energia elétrica', 'Gás', 'Internet', 'Telefone', 'Manutenção']],
    ['Alimentação', '#16a34a', '🛒', ['Supermercado', 'Açougue', 'Padaria', 'Restaurante', 'Delivery']],
    ['Transporte', '#f59e0b', '🚗', ['Combustível', 'Transporte público', 'Aplicativos de transporte', 'Manutenção do veículo', 'Seguro', 'Estacionamento']],
    ['Saúde', '#dc2626', '🩺', ['Plano de saúde', 'Consultas', 'Exames', 'Farmácia', 'Tratamentos']],
    ['Família', '#8b5cf6', '👨‍👩‍👧', ['Escola', 'Creche', 'Material escolar', 'Roupas', 'Brinquedos', 'Cuidados infantis']],
    ['Pessoal', '#ec4899', '🧴', ['Vestuário', 'Beleza', 'Academia', 'Lazer', 'Assinaturas', 'Cuidados pessoais']],
    ['Financeiro', '#0ea5e9', '🏦', ['Empréstimos', 'Financiamentos', 'Juros', 'Tarifas bancárias', 'Seguros']],
    ['Outros', '#64748b', '📦', ['Presentes', 'Viagens', 'Imprevistos', 'Doações', 'Despesas diversas']]
  ];
  grupos.forEach(function (g) {
    g[3].forEach(function (n) { lista.push({ tipo: 'despesa', grupo: g[0], nome: n, cor: g[1], icone: g[2] }); });
  });
  return lista.map(function (o) {
    return marcarCriacao_(Object.assign(o, { id: uuid_(), familia_id: fid, padrao: '1', ativo: '1' }), email);
  });
}

// ============================================================
// 9. PACOTE INICIAL E AUXILIARES DE CONSULTA
// ============================================================

function pacote_(c) {
  const fid = c.familia.id;
  return {
    versaoApp: VERSAO_APP,
    email: c.email,
    usuario: { id: c.usuario.id, nome: c.usuario.nome, email: c.usuario.email, perfil: c.usuario.perfil, editar_todos: c.usuario.editar_todos },
    familia: {
      id: fid, nome: c.familia.nome, moeda: c.familia.moeda || 'BRL',
      meta_economia_mensal: c.familia.meta_economia_mensal || '',
      base_comprometimento: c.familia.base_comprometimento || 'assumidas'
    },
    membros: membros_(fid),
    categorias: ativos_('categorias', fid).map(publico_),
    contas: ativos_('contas_bancarias', fid).map(publico_),
    cartoes: ativos_('cartoes', fid).map(publico_),
    sugestoes: sugestoes_(fid),
    urlApp: urlApp_(),
    urlPlanilha: c.perfil === 'admin' ? ss_().getUrl() : '',
    hoje: hoje_()
  };
}

function membros_(fid) {
  return tabela_('usuarios').rows
    .filter(function (r) { return r.familia_id === fid && r.excluido !== '1'; })
    .map(function (r) {
      return { id: r.id, nome: r.nome, email: r.email, perfil: r.perfil, status: r.status, editar_todos: r.editar_todos, ultimo_acesso: r.ultimo_acesso };
    });
}

/** Últimas descrições usadas → sugestão de categoria (nunca altera registros). */
function sugestoes_(fid) {
  const todos = ativos_('receitas', fid).map(function (r) { return { tipo: 'receita', r: r }; })
    .concat(ativos_('despesas', fid).map(function (r) { return { tipo: 'despesa', r: r }; }));
  todos.sort(function (a, b) { return String(b.r.criado_em).localeCompare(String(a.r.criado_em)); });
  const vistos = {}, out = [];
  for (let i = 0; i < todos.length && out.length < 300; i++) {
    const x = todos[i], k = x.tipo + '|' + String(x.r.descricao).toLowerCase();
    if (vistos[k]) continue;
    vistos[k] = 1;
    out.push({ tipo: x.tipo, descricao: x.r.descricao, categoria: x.r.categoria, subcategoria: x.r.subcategoria || '', responsavel: x.r.responsavel, conta_id: x.r.conta_id || '', forma_pagamento: x.r.forma_pagamento || '' });
  }
  return out;
}

function nomeDe_(email, fid) {
  const u = usuarioPorEmail_(String(email || '').toLowerCase(), fid);
  return u ? u.nome : email;
}

function ativos_(ent, fid) {
  return tabela_(ent).rows.filter(function (r) { return r.familia_id === fid && r.excluido !== '1'; });
}

function publico_(r) {
  const o = {};
  Object.keys(r).forEach(function (k) { if (k !== '_linha') o[k] = r[k]; });
  return o;
}

function resumo_(r) {
  const o = {};
  ['descricao', 'nome', 'valor', 'categoria', 'status', 'data', 'data_compra'].forEach(function (k) { if (r[k]) o[k] = r[k]; });
  return o;
}

function urlApp_() {
  try { return ScriptApp.getService().getUrl() || ''; } catch (e) { return ''; }
}

// ============================================================
// 10. ARMAZENAMENTO (Google Planilhas)
//     Toda leitura é em lote; escrita altera apenas as linhas envolvidas.
// ============================================================

function ss_() {
  if (_cache.ss) return _cache.ss;
  const props = PropertiesService.getScriptProperties();
  const id = props.getProperty('PLANILHA_ID');
  let ss = null;
  if (id) {
    try {
      ss = SpreadsheetApp.openById(id);
    } catch (e) {
      throw erro_('SEM_ACESSO_PLANILHA', 'Sua conta Google não tem acesso à planilha desta família. Peça ao administrador que envie um convite para este e-mail.');
    }
  } else {
    ss = SpreadsheetApp.getActiveSpreadsheet();
    if (!ss) throw erro_('NAO_INSTALADO', 'O sistema ainda não foi instalado. O administrador deve executar a função "instalar" no Apps Script.');
    props.setProperty('PLANILHA_ID', ss.getId());
  }
  _cache.ss = ss;
  return ss;
}

function colunas_(ent) {
  const d = ENTIDADES[ent];
  return d.cols.concat(d.semControle ? [] : COLS_CONTROLE);
}

function garantirEstrutura_(forcar) {
  const props = PropertiesService.getScriptProperties();
  if (!forcar && props.getProperty('ESTRUTURA') === VERSAO_ESTRUTURA) return;
  const ss = ss_();
  Object.keys(ENTIDADES).forEach(function (ent) { garantirAba_(ent, true); });
  ['Página1', 'Sheet1', 'Planilha1'].forEach(function (n) {
    const sh = ss.getSheetByName(n);
    if (sh && sh.getLastRow() === 0 && ss.getSheets().length > 1) ss.deleteSheet(sh);
  });
  props.setProperty('ESTRUTURA', VERSAO_ESTRUTURA);
}

/** Cria a aba se não existir; se existir, só acrescenta colunas que faltam (nunca apaga dados). */
function garantirAba_(ent, verificarCabecalho) {
  const ss = ss_(), def = ENTIDADES[ent];
  const cols = colunas_(ent);
  let sh = ss.getSheetByName(def.aba);
  if (!sh) {
    sh = ss.insertSheet(def.aba, ss.getSheets().length);
    // Texto puro em todas as células: evita que o Planilhas converta datas/valores pelo idioma.
    sh.getRange(1, 1, sh.getMaxRows(), cols.length).setNumberFormat('@');
    sh.getRange(1, 1, 1, cols.length).setValues([cols]).setFontWeight('bold').setBackground('#1e293b').setFontColor('#ffffff');
    sh.setFrozenRows(1);
    return sh;
  }
  if (verificarCabecalho) {
    const lc = Math.max(sh.getLastColumn(), 1);
    const atual = sh.getRange(1, 1, 1, lc).getValues()[0].map(String);
    const faltam = cols.filter(function (c) { return atual.indexOf(c) < 0; });
    if (faltam.length) {
      const inicio = atual.filter(Boolean).length ? lc + 1 : 1;
      sh.getRange(1, inicio, sh.getMaxRows(), faltam.length).setNumberFormat('@');
      sh.getRange(1, inicio, 1, faltam.length).setValues([faltam]).setFontWeight('bold').setBackground('#1e293b').setFontColor('#ffffff');
      sh.setFrozenRows(1);
    }
  }
  return sh;
}

/** Lê a aba inteira uma única vez por execução. Colunas mapeadas pelo nome do cabeçalho. */
function tabela_(ent) {
  const chave = 't_' + ent;
  if (_cache[chave]) return _cache[chave];
  const sh = garantirAba_(ent, false);
  const lastRow = sh.getLastRow(), lastCol = Math.max(sh.getLastColumn(), 1);
  const header = sh.getRange(1, 1, 1, lastCol).getValues()[0].map(function (h) { return String(h).trim(); });
  const valores = lastRow > 1 ? sh.getRange(2, 1, lastRow - 1, lastCol).getValues() : [];
  const rows = [];
  valores.forEach(function (linha, i) {
    const o = { _linha: i + 2 };
    let vazio = true;
    header.forEach(function (h, j) {
      if (!h) return;
      o[h] = norm_(linha[j]);
      if (o[h] !== '') vazio = false;
    });
    if (!vazio) {
      COLS_CONTROLE.forEach(function (k) { if (o[k] === undefined) o[k] = ''; });
      rows.push(o);
    }
  });
  const t = { ent: ent, sh: sh, header: header, rows: rows };
  _cache[chave] = t;
  return t;
}

function linhaDe_(t, obj) {
  return t.header.map(function (h) { return h ? (obj[h] == null ? '' : String(obj[h])) : ''; });
}

function garantirLinhas_(sh, ultima) {
  const max = sh.getMaxRows();
  if (ultima > max) {
    const add = Math.max(200, ultima - max);
    sh.insertRowsAfter(max, add);
    sh.getRange(max + 1, 1, add, Math.max(sh.getLastColumn(), 1)).setNumberFormat('@');
  }
}

function inserirVarios_(t, objs) {
  if (!objs.length) return;
  const ini = t.sh.getLastRow() + 1;
  garantirLinhas_(t.sh, ini + objs.length - 1);
  t.sh.getRange(ini, 1, objs.length, t.header.length).setNumberFormat('@')
    .setValues(objs.map(function (o) { return linhaDe_(t, o); }));
  objs.forEach(function (o, i) { o._linha = ini + i; t.rows.push(o); });
}

/** Atualiza SOMENTE a linha do registro. */
function atualizar_(t, obj) {
  t.sh.getRange(obj._linha, 1, 1, t.header.length).setNumberFormat('@').setValues([linhaDe_(t, obj)]);
}

function marcarCriacao_(obj, email) {
  const agora = agora_();
  return Object.assign(obj, {
    demo: obj.demo || '', excluido: '', criado_por: email, criado_em: agora,
    atualizado_por: email, atualizado_em: agora, versao: '1'
  });
}

function proximaVersao_(r) {
  return String((parseInt(r.versao, 10) || 1) + 1);
}

function auditar_(c, acao, ent, id, detalhes) {
  let det = '';
  try { det = JSON.stringify(detalhes || {}); } catch (e) { det = '{}'; }
  inserirVarios_(tabela_('historico'), [{
    id: uuid_(), familia_id: c.familia.id, data_hora: agora_(), usuario: c.email,
    acao: acao, entidade: ent, registro_id: id || '', detalhes: det.slice(0, 4000)
  }]);
}

/** Bloqueio global: duas pessoas nunca gravam ao mesmo tempo. */
function comLock_(fn) {
  const lock = LockService.getScriptLock();
  if (!lock.tryLock(25000)) throw erro_('OCUPADO', 'O sistema está gravando outra alteração. Tente novamente em instantes.');
  try {
    limparCacheTabelas_(); // relê os dados já dentro do bloqueio
    const r = fn();
    SpreadsheetApp.flush();
    return r;
  } finally {
    lock.releaseLock();
  }
}

function limparCacheTabelas_() {
  Object.keys(_cache).forEach(function (k) { if (k.indexOf('t_') === 0) delete _cache[k]; });
}

function marcarAlteracao_() {
  PropertiesService.getScriptProperties().setProperty('VERSAO_DADOS', String(Date.now()));
}

function versaoDados_() {
  return PropertiesService.getScriptProperties().getProperty('VERSAO_DADOS') || '0';
}

// ============================================================
// 11. UTILITÁRIOS
// ============================================================

function erro_(codigo, mensagem) {
  const e = new Error(mensagem);
  e.codigo = codigo;
  return e;
}

function uuid_() { return Utilities.getUuid(); }
function agora_() { return Utilities.formatDate(new Date(), FUSO, "yyyy-MM-dd'T'HH:mm:ss"); }
function hoje_() { return Utilities.formatDate(new Date(), FUSO, 'yyyy-MM-dd'); }

function norm_(v) {
  if (v instanceof Date) return Utilities.formatDate(v, FUSO, 'yyyy-MM-dd');
  if (v === null || v === undefined) return '';
  return String(v).trim();
}

function iso_(y, m, d) { return y + '-' + String(m).padStart(2, '0') + '-' + String(d).padStart(2, '0'); }
function ultimoDia_(y, m) { return new Date(Date.UTC(y, m, 0)).getUTCDate(); }
function addMes_(y, m, n) { const t = y * 12 + (m - 1) + n; return { y: Math.floor(t / 12), m: (t % 12 + 12) % 12 + 1 }; }
function addMesesDia_(iso, n) {
  const p = addMes_(+iso.slice(0, 4), +iso.slice(5, 7), n);
  return iso_(p.y, p.m, Math.min(+iso.slice(8, 10), ultimoDia_(p.y, p.m)));
}
function mesAno_(comp) {
  return ['jan', 'fev', 'mar', 'abr', 'mai', 'jun', 'jul', 'ago', 'set', 'out', 'nov', 'dez'][+comp.slice(5, 7) - 1] + '/' + comp.slice(2, 4);
}
function addDias_(iso, n) { const d = new Date(iso + 'T00:00:00Z'); d.setUTCDate(d.getUTCDate() + n); return d.toISOString().slice(0, 10); }
function diasEntre_(a, b) { return Math.round((Date.parse(b + 'T00:00:00Z') - Date.parse(a + 'T00:00:00Z')) / 86400000); }
function plural_(n, s, p) { return n + ' ' + (n === 1 ? s : p); }
function fmtPct_(n) { return String(n).replace('.', ',') + '%'; }
