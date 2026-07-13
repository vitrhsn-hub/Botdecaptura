# Botdecaptura
Bot de captura de dados web para automação das atividades de divida ativa, reintegração de posse e outras atividades

## primmeiro importamos as bibibliotecas e ferremanetas que serão usadas
import json
from bs4 import BeautifulSoup
from playwright.sync_api import sync_playwright

## essa etapa do codigo é a informações basicas para o bot realizar o login, url do site, usuario, senha, orgão que deseja efetuar login e processo alvo

    def capturar_resumo_processo():
        # 1. Configurações
        url_sei = "https://sei.rj.gov.br/"
        usuario = "Seu usuario"
        senha = "Sua senha"
        orgao_alvo = "****"  # Nome do Orgão exatamente como aparece na lista
        processo_alvo = "SEI-XXXXXX/XXXXXX/20XX"
        
## start do plawright para o bot acessar o pagina web 

    with sync_playwright() as p:
        # headless=False para você ver o navegador abrindo e selecionando as opções
        browser = p.chromium.launch(headless=False)
        context = browser.new_context()
        page = context.new_page()

        try:
            print("[*] Acessando a página do SEI-RJ...")
            page.goto(url_sei)

            print("[*] Preenchendo dados de login...")
            page.fill("id=txtUsuario", usuario)
            page.fill("id=pwdSenha", senha)

            # SELEÇÃO DO ÓRGÃO (Resolve o travamento)
            print(f"[*] Selecionando o órgão: {orgao_alvo}...")
            # O Playwright procura o texto visível na lista (label) e seleciona automaticamente
            page.select_option("id=selOrgao", label=orgao_alvo)

            # Clicar para entrar
            print("[*] Clicando no botão Acessar...")
            page.click("id=sbmAcessar")

            print(f"[*] Pesquisando o processo {processo_alvo}...")
            page.fill("id=txtPesquisaRapida", processo_alvo)
            page.keyboard.press("Enter")

            # 2. Acessar o Frame de Visualização
            print("[*] A aguardar o carregamento do processo...")
            frame_doc = page.frame_locator("frame[name='ifrVisualizacao'], iframe[name='ifrVisualizacao']")

            # NOVO PASSO: Clicar no botão "Consultar Andamento"
            print("[*] A clicar no botão 'Consultar Andamento'...")
            # O 'i' no final torna a busca insensível a maiúsculas/minúsculas
            botao_andamento = frame_doc.locator("[title*='Consultar Andamento' i]").first

            # Aguardamos que o botão fique visível e clicamos
            botao_andamento.wait_for(state="visible", timeout=15000)
            botao_andamento.click()

            # Agora sim, aguardamos que a tabela de andamentos carregue após o clique
            print("[*] A ler a tabela de andamentos do SEI...")
            frame_doc.locator("table.infraTable").first.wait_for(state="attached", timeout=30000)

            # 3. Extração do HTML para o BeautifulSoup
            html_iframe = frame_doc.locator("html").inner_html()
            soup = BeautifulSoup(html_iframe, "html.parser")

            # Procurar a tabela correta (garante que pegamos a tabela com "Data" no cabeçalho)
            tabela_andamentos = None
            for tb in soup.find_all("table", class_="infraTable"):
                cabecalhos = [th.text.strip().lower() for th in tb.find_all("th")]
                if any("data" in c for c in cabecalhos) and any("descri" in c for c in cabecalhos):
                    tabela_andamentos = tb
                    break

            if not tabela_andamentos:
                print("[-] Tabela de andamentos não encontrada dentro do iframe.")
                return

            # Mapeamento Dinâmico de Colunas (Evita erros se a ordem das colunas mudar no SEI)
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
            linhas = tabela_andamentos.find_all("tr")[1:]  # Pula a linha do cabeçalho
            andamentos_extraidos = []

            for linha in linhas:
                colunas = line = linha.find_all("td")
                if len(colunas) >= len(col_idx) and len(col_idx) > 0:
                    andamento = {
                        "data_hora": colunas[col_idx.get("data_hora", 0)].text.strip(),
                        "usuario": colunas[col_idx.get("usuario", 1)].text.strip(),
                        "setor": colunas[col_idx.get("setor", 2)].text.strip(),
                        "descricao": colunas[col_idx.get("descricao", 3)].text.strip(),
                    }
                    andamentos_extraidos.append(andamento)

            # 5. Exibir Resumo Solicitado
            total_andamentos = len(andamentos_extraidos)

            if total_andamentos > 0:
                # O SEI lista o andamento mais recente na última linha da lista [-1]
                ultimo_andamento = andamentos_extraidos[-1]

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
