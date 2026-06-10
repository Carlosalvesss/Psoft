Agora a saída do build confirma exatamente o que eu suspeitava, e dá pra resolver rápido. Olha o Build Output:

	•	UsuarioRepository.java → cannot find symbol class JpaRepository :6 — falta o import do JpaRepository (é a causa raiz).
	•	Usuario.java e ModelMapperConfig.java também aparecem com erro, em cascata.

Esses 10 erros vêm quase todos de imports faltando. Vamos corrigir os três arquivos.

1. repository/UsuarioRepository.java — adicione a linha do import:

import org.springframework.data.jpa.repository.JpaRepository;


2. config/ModelMapperConfig.java — provavelmente perdeu o import do ModelMapper. Confira que tem:

import org.modelmapper.ModelMapper;


3. model/Usuario.java — clica nele pra ver qual símbolo está vermelho. Provavelmente é algo de jakarta.persistence ou lombok. Me diz qual classe aparece em “cannot find symbol” que eu te dou o import exato.

O jeito mais rápido no IntelliJ é abrir cada arquivo com erro, clicar no símbolo vermelho e teclar Alt+Enter → Import class. Ele resolve sozinho.

Mas tem uma pergunta importante antes: esses imports não costumam sumir sozinhos. Se vários arquivos do projeto base perderam imports de uma vez (Spring, ModelMapper, JPA), pode ser que o Gradle não baixou/sincronizou as dependências — aí o IntelliJ não acha nenhuma biblioteca e “apaga” o reconhecimento.

Então faça também: clique no ícone do Gradle na lateral direita (ou Ctrl+Shift+O, ou botão direito no build.gradle → Reload Gradle Project) pra forçar o download das dependências.

Pra eu te dar a resposta certeira: expande os itens Usuario.java e ModelMapperConfig.java no Build Output (clica nas setinhas >) e me manda o texto do “cannot find symbol” de cada um. Com isso eu confirmo se é só import faltando ou se é o Gradle que não sincronizou.