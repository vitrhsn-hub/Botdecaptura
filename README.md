# Botdecaptura
Bot de captura de dados web para automação das atividades de divida ativa, reintegração de posse e outras atividades

### primmeiro importamos as bibibliotecas e ferremanetas que serão usadas 
     import json
     from bs4 import BeautifulSoup
     from playwright.sync_api import sync_playwright

### Essa etapa do código é onde será inserido as informações básicas para o bot realizar o login, url do site, usuário, senha, orgão que deseja efetuar login e processo alvo. (!!!!!essa parte por enquanto não é sigilosa, logo sua senha está exposta!!!!!!!!)
    def capturar_resumo_processo():
        # 1. Configurações
        url_sei = "https://sei.rj.gov.br/"
        usuario = "Seu usuario"
        senha = "Sua senha"
        orgao_alvo = "GOVXXXX"  # Nome do Orgão exatamente como aparece na lista
        processo_alvo = "SEI-XXXXXX/XXXXXX/20XX"
        
### Start do plawright para o bot acessar o pagina web anteriormente preenchida. 
    with sync_playwright() as p:
        # headless=False para você ver o navegador abrindo e selecionando as opções
        browser = p.chromium.launch(headless=False)
        context = browser.new_context()
        page = context.new_page()

### Bot incia a ação de acesso a pagina SEI e efetua o login prevviamente preenchido. 
        try:
            print("[*] Acessando a página do SEI-RJ...")
            page.goto(url_sei)

            print("[*] Preenchendo dados de login...")
            page.fill("id=txtUsuario", usuario)
            page.fill("id=pwdSenha", senha)

### Nessa etapa acontece a seleção do órgão para efetuar login.
            # SELEÇÃO DO ÓRGÃO 
            print(f"[*] Selecionando o órgão: {orgao_alvo}...")
            # O Playwright procura o texto visível na lista (label) e seleciona automaticamente
            page.select_option("id=selOrgao", label=orgao_alvo)

            # Clicar para entrar
            print("[*] Clicando no botão Acessar...")
            page.click("id=sbmAcessar")

### Assim que é realizado o login, o bot é orientado a OCULTAR MENU LATERAL para diminuir a margem de erro nos proximos passos.
            # OCULTAR MENU LATERAL
            print("[*] Ocultando o menu lateral do sistema...")
            botao_ocultar_menu = page.locator("img[title*='Ocultar Menu do Sistema']").first
            
### Próximo passo é a pesquisa do processo alvo que deve ser preenchido no inicio do codigo junto com os dados de login
            print(f"[*] Pesquisando o processo {processo_alvo}...")
            page.fill("id=txtPesquisaRapida", processo_alvo)
            page.keyboard.press("Enter")
             
### Após acessar o processo alvo, o bot acessará o andamento do processo, Aqui ele localiza onde está o botão "Consultar Andamento" por meio do código HTML da pagina web.
             # 2. Acessar o Frame da Árvore (Esquerda) para clicar no botão
            print("[*] Acessando a janela da árvore de documentos...")
            frame_arvore = page.frame(name="ifrArvore")

            if not frame_arvore:
                print("[-] Janela 'ifrArvore' não encontrada.")
                return

            # --- CLIQUE VIA JAVASCRIPT NO FRAME DA ÁRVORE ---
            print("[*] Executando o clique no botão 'Consultar Andamento'...")
            frame_arvore.evaluate("""() => {
                let botao = document.querySelector('#divConsultarAndamento a');
                if (!botao) {
                    const img = document.querySelector('img[alt="Consultar Andamento"]');
                    if (img) botao = img.parentElement;
                }
                if (botao) botao.click();
            }""")

### Realizando a Leitura da tabela que exibe o andamento e setores por onde o processo foi recebido.
            # 3. Mudar o foco para o Frame de Visualização (Direita) para ler a tabela
            print("[*] Lendo a tabela de andamentos na janela principal...")
            # Pequena pausa para garantir que o clique no lado esquerdo carregou o lado direito
            page.wait_for_timeout(3000)

            frame_visualizacao = page.frame(name="ifrVisualizacao")

            if not frame_visualizacao:
                print("[-] Janela 'ifrVisualizacao' não encontrada após o clique.")
                return

            frame_visualizacao.locator("table.infraTable").first.wait_for(state="attached", timeout=30000)

            # Extração do HTML para o BeautifulSoup
            html_iframe = frame_visualizacao.content()
            soup = BeautifulSoup(html_iframe, "html.parser")

            # Procurar a tabela correta com os dados
            tabela_andamentos = None
            for tb in soup.find_all("table", class_="infraTable"):
                cabecalhos = [th.text.strip().lower() for th in tb.find_all("th")]
                if any("data" in c for c in cabecalhos) and any("descri" in c for c in cabecalhos):
                    tabela_andamentos = tb
                    break

            if not tabela_andamentos:
                print("[-] Tabela de andamentos não encontrada dentro do iframe.")
                return

            # Mapeamento Dinâmico de Colunas
            col_idx = {}
            for i, th in enumerate(tabela_andamentos.find_all("th")):
                texto_th = th.text.strip().lower()
                if "data" in texto_th or "hora" in texto_th:
                    col_idx["data_hora"] = i
                elif "usuário" in texto_th or "usuario" in texto_th:
                    col_idx["usuario"] = i
                elif "unidade" in texto_th or "setor" in texto_th:
                    col_idx["setor"] = i
                elif "descri" in texto_th:
                    col_idx["descricao"] = i

            # 4. Ler todas as linhas de andamento
            linhas = tabela_andamentos.find_all("tr")[1:]
            andamentos_extraidos = []

            for linha in linhas:
                colunas = linha.find_all("td")
                if len(colunas) >= len(col_idx) and len(col_idx) > 0:
                    andamento = {
                        "data_hora": colunas[col_idx.get("data_hora", 0)].text.strip(),
                        "usuario": colunas[col_idx.get("usuario", 1)].text.strip(),
                        "setor": colunas[col_idx.get("setor", 2)].text.strip(),
                        "descricao": colunas[col_idx.get("descricao", 3)].text.strip(),
                    }
                    andamentos_extraidos.append(andamento)

### Exibe o resultado da pesquisa, mostrando o processo pesquisado, qunatidade de andamentos, último setor, último usuário e descrição, além de data e hora.
            # 5. Exibir Resumo Solicitado
            total_andamentos = len(andamentos_extraidos)

            if total_andamentos > 0:
                ultimo_andamento = andamentos_extraidos[0]

                print("\n==================================================")
                print(f" RESUMO DO PROCESSO: {processo_alvo}")
                print("==================================================")
                print(f"📊 Quantidade de andamentos: {total_andamentos}")
                print("\n📌 ÚLTIMO ANDAMENTO REGISTRADO:")
                print(f"   📅 Data e Hora: {ultimo_andamento['data_hora']}")
                print(f"   🏢 Setor:       {ultimo_andamento['setor']}")
                print(f"   👤 Usuário:     {ultimo_andamento['usuario']}")
                print(f"   📝 Descrição:   {ultimo_andamento['descricao']}")
                print("==================================================\n")

                # Salvar os dados em JSON
                with open("resumo_ultimo_andamento.json", "w", encoding="utf-8") as f:
                    json.dump({
                        "processo": processo_alvo,
                        "total_andamentos": total_andamentos,
                        "ultimo_andamento": ultimo_andamento
                    }, f, ensure_ascii=False, indent=4)
            else:
                print("[-] O processo não possui andamentos registrados na tabela.")

        except Exception as e:
            print(f"[-] Erro durante a automação: {e}")
        finally:
            print("[*] Encerrando o navegador...")
            browser.close()


if __name__ == "__main__":
    capturar_resumo_processo()
