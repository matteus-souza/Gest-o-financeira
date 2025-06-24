#Requires AutoHotkey v2.0
#SingleInstance Force
SetWorkingDir A_ScriptDir
FileEncoding "UTF-8"

global caminhoCSV := A_ScriptDir "\lancamentos.csv"
global descDesp, valorDesp, catDesp, descRec, valorRec, catRec
global boxRec, boxDesp, boxSaldo, listView

; --- Funções ---

SalvarDespesa(*) {
    tipo := "Despesa"
    desc := Trim(descDesp.Text)
    valor := StrReplace(Trim(valorDesp.Text), ",", ".")
    cat := catDesp.Text

    if (desc = "" || valor = "" || cat = "")
        return MsgBox("Preencha todos os campos da despesa.", "Erro", 48)
    if !IsNumber(valor)
        return MsgBox("Valor inválido. Use ponto como separador decimal.", "Erro", 48)

    data := FormatTime(A_Now, "dd/MM/yyyy HH:mm")
    FileAppend(tipo ";" desc ";" valor ";" cat ";" data "`n", caminhoCSV)

    MsgBox("Despesa salva com sucesso!", "✔", 64)
    descDesp.Text := ""
    valorDesp.Text := ""
    catDesp.Value := 1
    CarregarUltimosLancamentos()
    AtualizarResumo()
}

SalvarReceita(*) {
    tipo := "Receita"
    desc := Trim(descRec.Text)
    valor := StrReplace(Trim(valorRec.Text), ",", ".")
    cat := catRec.Text

    if (desc = "" || valor = "" || cat = "")
        return MsgBox("Preencha todos os campos da receita.", "Erro", 48)
    if !IsNumber(valor)
        return MsgBox("Valor inválido. Use ponto como separador decimal.", "Erro", 48)

    data := FormatTime(A_Now, "dd/MM/yyyy HH:mm")
    FileAppend(tipo ";" desc ";" valor ";" cat ";" data "`n", caminhoCSV)

    MsgBox("Receita salva com sucesso!", "✔", 64)
    descRec.Text := ""
    valorRec.Text := ""
    catRec.Value := 1
    CarregarUltimosLancamentos()
    AtualizarResumo()
}

ExcluirSelecionado(*) {
    linhaSelecionada := listView.GetNext()
    if (!linhaSelecionada)
        return MsgBox("Selecione um lançamento para excluir.", "Aviso", 48)

    if MsgBox("Tem certeza que deseja excluir o lançamento selecionado?", "Confirmar Exclusão", 36) != "Yes"
        return

    camposSelecionados := []
    Loop listView.GetCount("Column")
        camposSelecionados.Push(listView.GetText(linhaSelecionada, A_Index))

    linhasFinais := []
    conteudo := FileExist(caminhoCSV) ? FileRead(caminhoCSV) : ""
    linhas := StrSplit(Trim(conteudo, "`r`n"), "`n")

    for linha in linhas {
        campos := StrSplit(linha, ";")
        if (campos.Length = 5) {
            match := true
            Loop 5 {
                if (Trim(campos[A_Index]) != Trim(camposSelecionados[A_Index])) {
                    match := false
                    break
                }
            }
            if (!match)
                linhasFinais.Push(linha)
        }
    }

    FileDelete(caminhoCSV)
    for linha in linhasFinais
        FileAppend(linha "`n", caminhoCSV)

    MsgBox("Lançamento excluído com sucesso.", "OK", 64)
    CarregarUltimosLancamentos()
    AtualizarResumo()
}

CarregarUltimosLancamentos() {
    listView.Delete()
    if !FileExist(caminhoCSV)
        return

    conteudo := FileRead(caminhoCSV)
    linhas := StrSplit(Trim(conteudo, "`r`n"), "`n")
    ultimas := (linhas.Length <= 10) ? linhas : linhas.Slice(-10)

    for linha in ultimas {
        campos := StrSplit(linha, ";")
        if (campos.Length = 5)
            listView.Add(, campos*)
    }
}

AtualizarResumo() {
    totalRec := 0.0
    totalDesp := 0.0

    if FileExist(caminhoCSV) {
        conteudo := FileRead(caminhoCSV)
        linhas := StrSplit(Trim(conteudo, "`r`n"), "`n")

        for linha in linhas {
            campos := StrSplit(linha, ";")
            if (campos.Length = 5) {
                tipo := campos[1]
                valor := campos[3]
                if IsNumber(valor) {
                    if (tipo = "Receita")
                        totalRec += valor
                    else if (tipo = "Despesa")
                        totalDesp += valor
                }
            }
        }
    }

    saldo := totalRec - totalDesp
    boxRec.Text := Format("{:.2f}", totalRec)
    boxDesp.Text := Format("{:.2f}", totalDesp)
    boxSaldo.Text := Format("{:.2f}", saldo)
}

AbrirCSV(*) {
    if FileExist(caminhoCSV)
        Run(caminhoCSV)
    else
        MsgBox("Nenhum dado foi registrado ainda.", "Aviso", 48)
}

; --- Interface ---
janela := Gui()
janela.Title := "Gestor Financeiro"
abas := janela.Add("Tab3", "w600", ["Despesas", "Receitas", "Resumo"])

; --- Abas ---
abas.UseTab(1)
janela.Add("Text",, "Descrição:")
descDesp := janela.Add("Edit", "w300")
janela.Add("Text",, "Valor (R$):")
valorDesp := janela.Add("Edit", "w100")
janela.Add("Text",, "Categoria:")
catDesp := janela.Add("DropDownList", "w200", ["Alimentação", "Transporte", "Lazer", "Saúde", "Outros"])
janela.Add("Button",, "💾 Salvar Despesa").OnEvent("Click", SalvarDespesa)

abas.UseTab(2)
janela.Add("Text",, "Descrição:")
descRec := janela.Add("Edit", "w300")
janela.Add("Text",, "Valor (R$):")
valorRec := janela.Add("Edit", "w100")
janela.Add("Text",, "Categoria:")
catRec := janela.Add("DropDownList", "w200", ["Salário", "Venda", "Presente", "Outros"])
janela.Add("Button",, "💾 Salvar Receita").OnEvent("Click", SalvarReceita)

abas.UseTab(3)
janela.Add("Text",, "💰 Total de Receitas:")
boxRec := janela.Add("Edit", "ReadOnly w120")
janela.Add("Text",, "💸 Total de Despesas:")
boxDesp := janela.Add("Edit", "ReadOnly w120")
janela.Add("Text",, "⚖️ Saldo Atual:")
boxSaldo := janela.Add("Edit", "ReadOnly w120")

abas.UseTab()
janela.Add("Text",, "Últimos lançamentos:")
listView := janela.Add("ListView", "w580 r10", ["Tipo", "Descrição", "Valor", "Categoria", "Data/Hora"])
janela.Add("Button",, "📂 Abrir CSV").OnEvent("Click", AbrirCSV)
janela.Add("Button",, "🗑️ Excluir Selecionado").OnEvent("Click", ExcluirSelecionado)
janela.Add("Button",, "❌ Sair").OnEvent("Click", (*) => ExitApp())

janela.Show()
CarregarUltimosLancamentos()
AtualizarResumo()