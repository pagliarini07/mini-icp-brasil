# Atividade — Sistemas ICP 

**Uso educacional.** Não utilize em produção. Emissões reais devem seguir as regras da ICP-Brasil com AC credenciada.

---

## Ajustes desta versão
- Usuários padrão: **luis**, **monica** e **gustavo** (FakeCA).
- Salva **CSRs** da Intermediária e dos Usuários.
- Assinatura com extensão **`.assinatura`** + arquivo **`.sha256`**.
- Aceita **CPF/CNPJ com pontuação** (normaliza internamente).
- Datas *timezone-aware* (evitam DeprecationWarnings).

---

## Objetivo (o que será demonstrado)
1. **CA Raiz** (autoassinada) e **CA Intermediária** (pathlen=0).  
2. Emissão de certificados de **usuário final** (e-CPF simulado).  
3. **Assinatura** e **verificação** de documento com **cadeia confiável**.  
4. **Rejeição** de certificado emitido por **FakeCA** (cadeia não confiável).  
5. (Opcional) **Revogação/CRL** e verificação com CRL → deve **falhar**.

---

## Requisitos
- Windows + PowerShell  
- Python **3.11+**  
- Biblioteca Python:
  ~~~powershell
  py -m pip install --user cryptography
  ~~~

---

## Passo a passo (PowerShell)

Execute os comandos **na pasta do projeto** (onde existe `pki_py\pki.py`).

~~~powershell
cd <pasta-extraida>  # ex.: C:\Users\...\Documents\atividade-sistemas-icp
~~~

### 1) Estrutura e CAs
~~~powershell
py pki_py\pki.py init
py pki_py\pki.py gen-root
py pki_py\pki.py gen-intermediate
~~~

### 2) Usuários (e-CPF simulado)
~~~powershell
py pki_py\pki.py gen-user --name luis   --cn "Luis da Silva"   --email luis@example.com   --cpf "123.456.789-00"
py pki_py\pki.py gen-user --name monica --cn "Monica Oliveira" --email monica@example.com --cpf "987.654.321-00"
~~~
> O **subjectAltName** inclui e-mail e o **OID do CPF** (2.16.76.1.3.1).

### 3) Assinar e verificar (cadeia **válida** – Luis)
~~~powershell
New-Item -ItemType Directory -Force .\docs | Out-Null
Set-Content -Path .\docs\contrato.txt -Value "Contrato - Exemplo (Luis/Monica/Gustavo)"

py pki_py\pki.py sign --file docs\contrato.txt --key pki\usuarios\luis\private\luis.key.pem

py pki_py\pki.py verify --file docs\contrato.txt `
  --sig docs\contrato.txt.assinatura `
  --cert  pki\usuarios\luis\certs\luis.crt `
  --chain pki\ca-intermediaria\certs\intermediateCA.crt `
  --root  pki\raiz-ca\certs\rootCA.crt
~~~

**Saída esperada:**  
`[OK] Assinatura válida e cadeia confiável.`

### 4) FakeCA (cadeia **falsa**, deve ser **rejeitada** – Gustavo)
~~~powershell
py pki_py\pki.py fakeca
py pki_py\pki.py sign --file docs\contrato.txt --key pki\usuarios\gustavo\private\gustavo.key.pem
Copy-Item docs\contrato.txt.assinatura docs\contrato.txt.assinatura.gustavo

py pki_py\pki.py verify --file docs\contrato.txt `
  --sig docs\contrato.txt.assinatura.gustavo `
  --cert  pki\usuarios\gustavo\certs\gustavo.crt `
  --chain pki\ca-intermediaria\certs\intermediateCA.crt `
  --root  pki\raiz-ca\certs\rootCA.crt
~~~

**Saída esperada:**  
`Falha de verificação: usuário não foi assinado pela intermediária informada.`

### 5) (Opcional) Revogação/CRL (deve **falhar** por revogação – Luis)
> Gere nova assinatura com **Luis** e depois verifique usando **CRL**.
~~~powershell
# Revogar Luis e gerar CRL
py pki_py\pki.py revoke --user luis

# Reassinar com Luis (garante que assinatura e certificado correspondem)
py pki_py\pki.py sign --file docs\contrato.txt --key pki\usuarios\luis\private\luis.key.pem
Copy-Item docs\contrato.txt.assinatura docs\contrato.txt.assinatura.luis.novo

# Verificar com CRL (deve falhar)
py pki_py\pki.py verify --file docs\contrato.txt `
  --sig  docs\contrato.txt.assinatura.luis.novo `
  --cert pki\usuarios\luis\certs\luis.crt `
  --chain pki\ca-intermediaria\certs\intermediateCA.crt `
  --root pki\raiz-ca\certs\rootCA.crt `
  --crl  pki\ca-intermediaria\crl\intermediateCA.crl.pem
~~~

**Saída esperada:**  
`Falha de verificação: certificado do usuário está REVOGADO (CRL).`

### 6) Execução automática (gera um log de referência)
~~~powershell
py pki_py\pki.py demo *> docs\execucao-luis-monica-gustavo.log
~~~

---

## Estrutura gerada (esperada)

/pki/
├─ raiz-ca/
│ ├─ private/ (rootCA.key.pem)
│ ├─ certs/ (rootCA.crt)
│ └─ crl/
├─ ca-intermediaria/
│ ├─ private/ (intermediateCA.key.pem)
│ ├─ certs/ (intermediateCA.crt, ca-chain.pem)
│ ├─ csr/ (intermediate.csr)
│ └─ crl/ (intermediateCA.crl.pem) # se usar revogação
└─ usuarios/
├─ luis/
│ ├─ private/ (luis.key.pem)
│ ├─ csr/ (luis.csr)
│ └─ certs/ (luis.crt, ca-chain.pem)
├─ monica/
└─ gustavo/ (emitido pela FakeCA)

## Evidências esperadas (mensagens-chave)
- **Cadeia válida (Luis):**  
  `[OK] Assinatura válida e cadeia confiável.`
- **FakeCA (Gustavo) – cadeia não confiável:**  
  `Falha de verificação: usuário não foi assinado pela intermediária informada.`
- **Revogação/CRL (Luis):**  
  `Falha de verificação: certificado do usuário está REVOGADO (CRL).`

  ## Solução de problemas
- **“Assinatura inválida do arquivo”**  
  Assinou com um usuário e verificou com o certificado de outro, **ou** regenerou as chaves (ex.: rodou `demo`) depois de assinar.  
  **Correção:** refaça a assinatura com a **mesma chave** do certificado usado na verificação.

- **`can't open file ... pki_py\pki.py`**  
  Não está na pasta correta.  
  **Correção:** `Test-Path .\pki_py\pki.py` deve retornar **True**.

- **`No module named cryptography`**  
  **Correção:** `py -m pip install --user cryptography`.

- **PowerShell bloqueando venv (`Activate.ps1`)**  
  **Correção rápida (só nesta sessão):**  
  `Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force`

---

## Explicação técnica 
- **Raiz**: X.509 autoassinada, `basicConstraints=CA:true`, `keyUsage=keyCertSign,cRLSign`.  
- **Intermediária**: assinada pela raiz, `basicConstraints=CA:true,pathlen=0`, `keyUsage=keyCertSign,cRLSign`.  
- **Usuário**: não-CA, `keyUsage=digitalSignature,keyEncipherment`, EKU para **ClientAuth/EmailProtection**; `subjectAltName` com **e-mail** e **CPF (OID 2.16.76.1.3.1)**.  
- **Assinatura**: RSA + SHA-256 (PKCS#1 v1.5).  
- **Verificação**: confere hash/assinatura e **cadeia** (`usuário ← intermediária ← raiz`), com checagem opcional de **CRL**.

---
