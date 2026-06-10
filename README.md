UsuarioServiceImpl


package com.ufcg.psoft.project.service.usuario;

import com.ufcg.psoft.project.exception.UsuarioNaoExisteException;
import com.ufcg.psoft.project.exception.CodigoDeAcessoInvalidoException;
import com.ufcg.psoft.project.model.Usuario;
import com.ufcg.psoft.project.repository.UsuarioRepository;
import com.ufcg.psoft.project.service.historico.HistoricoService;
import com.ufcg.psoft.project.dto.UsuarioPostPutRequestDTO;
import com.ufcg.psoft.project.dto.UsuarioResponseDTO;
import org.modelmapper.ModelMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.List;
import java.util.stream.Collectors;

@Service
public class UsuarioServiceImpl implements UsuarioService {

    @Autowired
    UsuarioRepository usuarioRepository;
    @Autowired
    ModelMapper modelMapper;
    @Autowired
    HistoricoService historicoService;

    @Override
    @Transactional
    public UsuarioResponseDTO alterar(Long id, String codigoAcesso, UsuarioPostPutRequestDTO usuarioPostPutRequestDTO) {
        Usuario usuario = usuarioRepository.findById(id).orElseThrow(UsuarioNaoExisteException::new);
        if (!usuario.getCodigo().equals(codigoAcesso)) {
            throw new CodigoDeAcessoInvalidoException();
        }

        // Captura os valores ANTES de o modelMapper sobrescrever a entidade.
        String enderecoAnterior = usuario.getEndereco();
        String codigoAnterior = usuario.getCodigo();
        LocalDateTime dataHora = LocalDateTime.now();

        // US01: so endereco e codigo sao rastreados. Registro so e criado se houve mudanca.
        historicoService.registrarSeAlterado(usuario, "endereco", enderecoAnterior, usuarioPostPutRequestDTO.getEndereco(), dataHora);
        historicoService.registrarSeAlterado(usuario, "codigo", codigoAnterior, usuarioPostPutRequestDTO.getCodigo(), dataHora);

        modelMapper.map(usuarioPostPutRequestDTO, usuario);
        usuarioRepository.save(usuario);
        return modelMapper.map(usuario, UsuarioResponseDTO.class);
    }

    @Override
    public UsuarioResponseDTO criar(UsuarioPostPutRequestDTO usuarioPostPutRequestDTO) {
        Usuario usuario = modelMapper.map(usuarioPostPutRequestDTO, Usuario.class);
        usuarioRepository.save(usuario);
        return modelMapper.map(usuario, UsuarioResponseDTO.class);
    }

    @Override
    public void remover(Long id, String codigoAcesso) {
        Usuario usuario = usuarioRepository.findById(id).orElseThrow(UsuarioNaoExisteException::new);
        if (!usuario.getCodigo().equals(codigoAcesso)) {
            throw new CodigoDeAcessoInvalidoException();
        }
        usuarioRepository.delete(usuario);
    }

    @Override
    public List<UsuarioResponseDTO> listar() {
        List<Usuario> usuarios = usuarioRepository.findAll();
        return usuarios.stream()
                .map(UsuarioResponseDTO::new)
                .collect(Collectors.toList());
    }

    @Override
    public UsuarioResponseDTO recuperar(Long id) {
        Usuario usuario = usuarioRepository.findById(id).orElseThrow(UsuarioNaoExisteException::new);
        return new UsuarioResponseDTO(usuario);
    }
}





HistoricoAlteracao


package com.ufcg.psoft.project.model;

import com.fasterxml.jackson.annotation.JsonProperty;
import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Entity
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class HistoricoAlteracao {

    @JsonProperty("id")
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;

    @JsonProperty("usuarioId")
    @Column(nullable = false)
    private Long usuarioId;

    @JsonProperty("campoAlterado")
    @Column(nullable = false)
    private String campoAlterado;

    @JsonProperty("valorAnterior")
    @Column
    private String valorAnterior;

    @JsonProperty("valorNovo")
    @Column
    private String valorNovo;

    @JsonProperty("dataHora")
    @Column(nullable = false)
    private LocalDateTime dataHora;
}




HistoricoAlteracaoRepository


package com.ufcg.psoft.project.repository;

import com.ufcg.psoft.project.model.HistoricoAlteracao;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface HistoricoAlteracaoRepository extends JpaRepository<HistoricoAlteracao, Long> {

    // US02: registros de um usuario, em ordem cronologica (id como desempate estavel)
    List<HistoricoAlteracao> findByUsuarioIdOrderByDataHoraAscIdAsc(Long usuarioId);

    // US03: todos os registros, em ordem cronologica
    List<HistoricoAlteracao> findAllByOrderByDataHoraAscIdAsc();
}




HistoricoAlteracaoResponseDTO


package com.ufcg.psoft.project.dto;

import com.fasterxml.jackson.annotation.JsonProperty;
import com.ufcg.psoft.project.model.HistoricoAlteracao;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class HistoricoAlteracaoResponseDTO {

    @JsonProperty("id")
    private Long id;

    @JsonProperty("usuarioId")
    private Long usuarioId;

    @JsonProperty("campoAlterado")
    private String campoAlterado;

    @JsonProperty("valorAnterior")
    private String valorAnterior;

    @JsonProperty("valorNovo")
    private String valorNovo;

    @JsonProperty("dataHora")
    private LocalDateTime dataHora;

    public HistoricoAlteracaoResponseDTO(HistoricoAlteracao historico) {
        this.id = historico.getId();
        this.usuarioId = historico.getUsuarioId();
        this.campoAlterado = historico.getCampoAlterado();
        this.valorAnterior = historico.getValorAnterior();
        this.valorNovo = historico.getValorNovo();
        this.dataHora = historico.getDataHora();
    }
}




HistoricoService



package com.ufcg.psoft.project.service.historico;

import com.ufcg.psoft.project.dto.HistoricoAlteracaoResponseDTO;
import com.ufcg.psoft.project.model.Usuario;

import java.time.LocalDateTime;
import java.util.List;

public interface HistoricoService {

    /**
     * US01: registra a alteracao de um campo. A regra "nenhum registro quando os
     * dados sao identicos" fica centralizada aqui: se valorAnterior == valorNovo,
     * nada e gravado.
     */
    void registrarSeAlterado(Usuario usuario, String campo, String valorAnterior, String valorNovo, LocalDateTime dataHora);

    // US02: historico de um usuario especifico, em ordem cronologica
    List<HistoricoAlteracaoResponseDTO> listarPorUsuario(Long usuarioId);

    // US03: historico geral de todos os usuarios, em ordem cronologica
    List<HistoricoAlteracaoResponseDTO> listarTodos();
}



HistoricoServiceImpl


package com.ufcg.psoft.project.service.historico;

import com.ufcg.psoft.project.dto.HistoricoAlteracaoResponseDTO;
import com.ufcg.psoft.project.exception.UsuarioNaoExisteException;
import com.ufcg.psoft.project.model.HistoricoAlteracao;
import com.ufcg.psoft.project.model.Usuario;
import com.ufcg.psoft.project.repository.HistoricoAlteracaoRepository;
import com.ufcg.psoft.project.repository.UsuarioRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Objects;
import java.util.stream.Collectors;

@Service
public class HistoricoServiceImpl implements HistoricoService {

    @Autowired
    HistoricoAlteracaoRepository historicoRepository;
    @Autowired
    UsuarioRepository usuarioRepository;

    @Override
    public void registrarSeAlterado(Usuario usuario, String campo, String valorAnterior, String valorNovo, LocalDateTime dataHora) {
        // Nenhum registro quando os dados informados sao identicos aos ja armazenados
        if (Objects.equals(valorAnterior, valorNovo)) {
            return;
        }
        HistoricoAlteracao historico = HistoricoAlteracao.builder()
                .usuarioId(usuario.getId())
                .campoAlterado(campo)
                .valorAnterior(valorAnterior)
                .valorNovo(valorNovo)
                .dataHora(dataHora)
                .build();
        historicoRepository.save(historico);
    }

    @Override
    public List<HistoricoAlteracaoResponseDTO> listarPorUsuario(Long usuarioId) {
        // Valida a existencia do usuario, coerente com o restante da API.
        // "Lista vazia" se refere a um usuario existente que ainda nao teve alteracoes.
        usuarioRepository.findById(usuarioId).orElseThrow(UsuarioNaoExisteException::new);
        return historicoRepository.findByUsuarioIdOrderByDataHoraAscIdAsc(usuarioId)
                .stream()
                .map(HistoricoAlteracaoResponseDTO::new)
                .collect(Collectors.toList());
    }

    @Override
    public List<HistoricoAlteracaoResponseDTO> listarTodos() {
        return historicoRepository.findAllByOrderByDataHoraAscIdAsc()
                .stream()
                .map(HistoricoAlteracaoResponseDTO::new)
                .collect(Collectors.toList());
    }
}


HistoricoController


package com.ufcg.psoft.project.controller;

import com.ufcg.psoft.project.service.historico.HistoricoService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping(
        value = "/historico",
        produces = MediaType.APPLICATION_JSON_VALUE
)
public class HistoricoController {

    @Autowired
    HistoricoService historicoService;

    // US03: historico geral de alteracoes
    @GetMapping("")
    public ResponseEntity<?> listarHistoricoGeral() {
        return ResponseEntity
                .status(HttpStatus.OK)
                .body(historicoService.listarTodos());
    }

    // US02: historico de um usuario especifico
    @GetMapping("/usuario/{usuarioId}")
    public ResponseEntity<?> listarHistoricoUsuario(
            @PathVariable Long usuarioId) {
        return ResponseEntity
                .status(HttpStatus.OK)
                .body(historicoService.listarPorUsuario(usuarioId));
    }
}



HistoricoAlteracaoTest


package com.ufcg.psoft.project;

import com.ufcg.psoft.project.dto.HistoricoAlteracaoResponseDTO;
import com.ufcg.psoft.project.dto.UsuarioPostPutRequestDTO;
import com.ufcg.psoft.project.exception.UsuarioNaoExisteException;
import com.ufcg.psoft.project.model.Usuario;
import com.ufcg.psoft.project.repository.HistoricoAlteracaoRepository;
import com.ufcg.psoft.project.repository.UsuarioRepository;
import com.ufcg.psoft.project.service.historico.HistoricoService;
import com.ufcg.psoft.project.service.usuario.UsuarioService;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@DisplayName("Historico de alteracoes de usuarios (US01, US02, US03)")
class HistoricoAlteracaoTest {

    @Autowired
    UsuarioService usuarioService;
    @Autowired
    HistoricoService historicoService;
    @Autowired
    UsuarioRepository usuarioRepository;
    @Autowired
    HistoricoAlteracaoRepository historicoRepository;

    Usuario usuario;

    @BeforeEach
    void setUp() {
        usuario = usuarioRepository.save(
                Usuario.builder()
                        .nome("Joao")
                        .endereco("Rua A, 100")
                        .codigo("123456")
                        .build()
        );
    }

    @AfterEach
    void tearDown() {
        historicoRepository.deleteAll();
        usuarioRepository.deleteAll();
    }

    private UsuarioPostPutRequestDTO dto(String nome, String endereco, String codigo) {
        return UsuarioPostPutRequestDTO.builder()
                .nome(nome)
                .endereco(endereco)
                .codigo(codigo)
                .build();
    }

    @Test
    @DisplayName("US01: alterar o endereco cria um registro de historico")
    void alterarEnderecoCriaRegistro() {
        usuarioService.alterar(usuario.getId(), "123456", dto("Joao", "Rua B, 200", "123456"));

        List<HistoricoAlteracaoResponseDTO> historico = historicoService.listarPorUsuario(usuario.getId());
        assertEquals(1, historico.size());
        HistoricoAlteracaoResponseDTO registro = historico.get(0);
        assertAll(
                () -> assertEquals("endereco", registro.getCampoAlterado()),
                () -> assertEquals("Rua A, 100", registro.getValorAnterior()),
                () -> assertEquals("Rua B, 200", registro.getValorNovo()),
                () -> assertEquals(usuario.getId(), registro.getUsuarioId()),
                () -> assertNotNull(registro.getDataHora())
        );
    }

    @Test
    @DisplayName("US01: alterar o codigo cria um registro de historico")
    void alterarCodigoCriaRegistro() {
        usuarioService.alterar(usuario.getId(), "123456", dto("Joao", "Rua A, 100", "654321"));

        List<HistoricoAlteracaoResponseDTO> historico = historicoService.listarPorUsuario(usuario.getId());
        assertEquals(1, historico.size());
        assertAll(
                () -> assertEquals("codigo", historico.get(0).getCampoAlterado()),
                () -> assertEquals("123456", historico.get(0).getValorAnterior()),
                () -> assertEquals("654321", historico.get(0).getValorNovo())
        );
    }

    @Test
    @DisplayName("US01: alterar codigo e endereco cria dois registros")
    void alterarCodigoEEnderecoCriaDoisRegistros() {
        usuarioService.alterar(usuario.getId(), "123456", dto("Joao", "Rua B, 200", "654321"));

        assertEquals(2, historicoService.listarPorUsuario(usuario.getId()).size());
    }

    @Test
    @DisplayName("US01: dados identicos nao criam registro")
    void dadosIdenticosNaoCriamRegistro() {
        usuarioService.alterar(usuario.getId(), "123456", dto("Joao Silva", "Rua A, 100", "123456"));

        assertTrue(historicoService.listarPorUsuario(usuario.getId()).isEmpty());
    }

    @Test
    @DisplayName("US01: alterar apenas o nome nao cria registro")
    void alterarApenasNomeNaoCriaRegistro() {
        usuarioService.alterar(usuario.getId(), "123456", dto("Novo Nome", "Rua A, 100", "123456"));

        assertTrue(historicoService.listarPorUsuario(usuario.getId()).isEmpty());
    }

    @Test
    @DisplayName("US02: usuario sem alteracoes retorna lista vazia")
    void usuarioSemAlteracoesRetornaListaVazia() {
        assertTrue(historicoService.listarPorUsuario(usuario.getId()).isEmpty());
    }

    @Test
    @DisplayName("US02: retorna apenas os registros do usuario informado, em ordem cronologica")
    void retornaApenasRegistrosDoUsuarioEmOrdem() {
        Usuario outro = usuarioRepository.save(
                Usuario.builder().nome("Maria").endereco("Rua C, 300").codigo("000000").build()
        );

        usuarioService.alterar(usuario.getId(), "123456", dto("Joao", "Rua B, 200", "123456")); // muda endereco
        usuarioService.alterar(usuario.getId(), "123456", dto("Joao", "Rua B, 200", "111111")); // muda codigo
        usuarioService.alterar(outro.getId(), "000000", dto("Maria", "Rua D, 400", "000000"));   // outro usuario

        List<HistoricoAlteracaoResponseDTO> historico = historicoService.listarPorUsuario(usuario.getId());
        assertEquals(2, historico.size());
        assertTrue(historico.stream().allMatch(h -> h.getUsuarioId().equals(usuario.getId())));
        assertEquals("endereco", historico.get(0).getCampoAlterado());
        assertEquals("codigo", historico.get(1).getCampoAlterado());
    }

    @Test
    @DisplayName("US02: consultar historico de usuario inexistente lanca excecao")
    void historicoUsuarioInexistenteLancaExcecao() {
        assertThrows(UsuarioNaoExisteException.class,
                () -> historicoService.listarPorUsuario(999999L));
    }

    @Test
    @DisplayName("US03: historico geral retorna registros de todos os usuarios")
    void historicoGeralRetornaTodos() {
        Usuario outro = usuarioRepository.save(
                Usuario.builder().nome("Maria").endereco("Rua C, 300").codigo("000000").build()
        );
        usuarioService.alterar(usuario.getId(), "123456", dto("Joao", "Rua B, 200", "123456"));
        usuarioService.alterar(outro.getId(), "000000", dto("Maria", "Rua D, 400", "000000"));

        assertEquals(2, historicoService.listarTodos().size());
    }

    @Test
    @DisplayName("US03: sem alteracoes registradas retorna lista vazia")
    void historicoGeralVazio() {
        assertTrue(historicoService.listarTodos().isEmpty());
    }
}



























testes

package com.ufcg.psoft.project.controller;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.ufcg.psoft.project.dto.UsuarioPostPutRequestDTO;
import com.ufcg.psoft.project.dto.UsuarioResponseDTO;
import com.ufcg.psoft.project.exception.CustomErrorType;
import com.ufcg.psoft.project.model.Usuario;
import com.ufcg.psoft.project.repository.UsuarioRepository;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.util.Arrays;
import java.util.List;

import static org.junit.jupiter.api.Assertions.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
@DisplayName("Testes do controlador de Usuários")
public class UsuarioControllerTests {

    final String URI_USUARIOS = "/usuario";

    @Autowired
    MockMvc driver;

    @Autowired
    UsuarioRepository usuarioRepository;

    ObjectMapper objectMapper = new ObjectMapper();

    Usuario usuario;

    UsuarioPostPutRequestDTO usuarioPostPutRequestDTO;

    @BeforeEach
    void setup() {
        // Object Mapper suporte para LocalDateTime
        objectMapper.registerModule(new JavaTimeModule());
        usuario = usuarioRepository.save(Usuario.builder()
                .nome("usuario Um da Silva")
                .endereco("Rua dos Testes, 123")
                .codigo("123456")
                .build()
        );
        usuarioPostPutRequestDTO = UsuarioPostPutRequestDTO.builder()
                .nome(usuario.getNome())
                .endereco(usuario.getEndereco())
                .codigo(usuario.getCodigo())
                .build();
    }

    @AfterEach
    void tearDown() {
        usuarioRepository.deleteAll();
    }

    @Nested
    @DisplayName("Conjunto de casos de verificação de nome")
    class usuarioVerificacaoNome {

        @Test
        @DisplayName("Quando recuperamos um usuario com dados válidos")
        void quandoRecuperamosNomeDousuarioValido() throws Exception {

            // Act
            String responseJsonString = driver.perform(get(URI_USUARIOS + "/" + usuario.getId()))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            Usuario resultado = objectMapper.readValue(responseJsonString, Usuario.UsuarioBuilder.class).build();

            // Assert
            assertEquals("usuario Um da Silva", resultado.getNome());
        }

        @Test
        @DisplayName("Quando alteramos o nome do usuario com dados válidos")
        void quandoAlteramosNomeDousuarioValido() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setNome("usuario Um Alterado");

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            Usuario resultado = objectMapper.readValue(responseJsonString, Usuario.UsuarioBuilder.class).build();

            // Assert
            assertEquals("usuario Um Alterado", resultado.getNome());
        }

        @Test
        @DisplayName("Quando alteramos o nome do usuario nulo")
        void quandoAlteramosNomeDousuarioNulo() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setNome(null);

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isBadRequest())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("Erros de validacao encontrados", resultado.getMessage()),
                    () -> assertEquals("Nome obrigatorio", resultado.getErrors().get(0))
            );
        }

        @Test
        @DisplayName("Quando alteramos o nome do usuario vazio")
        void quandoAlteramosNomeDousuarioVazio() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setNome("");

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isBadRequest())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("Erros de validacao encontrados", resultado.getMessage()),
                    () -> assertEquals("Nome obrigatorio", resultado.getErrors().get(0))
            );
        }
    }

    @Nested
    @DisplayName("Conjunto de casos de verificação do endereço")
    class usuarioVerificacaoEndereco {

        @Test
        @DisplayName("Quando alteramos o endereço do usuario com dados válidos")
        void quandoAlteramosEnderecoDousuarioValido() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setEndereco("Endereco Alterado");

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            UsuarioResponseDTO resultado = objectMapper.readValue(responseJsonString, UsuarioResponseDTO.UsuarioResponseDTOBuilder.class).build();

            // Assert
            assertEquals("Endereco Alterado", resultado.getEndereco());
        }

        @Test
        @DisplayName("Quando alteramos o endereço do usuario nulo")
        void quandoAlteramosEnderecoDousuarioNulo() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setEndereco(null);

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isBadRequest())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("Erros de validacao encontrados", resultado.getMessage()),
                    () -> assertEquals("Endereco obrigatorio", resultado.getErrors().get(0))
            );
        }

        @Test
        @DisplayName("Quando alteramos o endereço do usuario vazio")
        void quandoAlteramosEnderecoDousuarioVazio() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setEndereco("");

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isBadRequest())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("Erros de validacao encontrados", resultado.getMessage()),
                    () -> assertEquals("Endereco obrigatorio", resultado.getErrors().get(0))
            );
        }
    }

    @Nested
    @DisplayName("Conjunto de casos de verificação do código de acesso")
    class usuarioVerificacaoCodigoAcesso {

        @Test
        @DisplayName("Quando alteramos o código de acesso do usuario nulo")
        void quandoAlteramosCodigoAcessoDousuarioNulo() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setCodigo(null);

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isBadRequest())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("Erros de validacao encontrados", resultado.getMessage()),
                    () -> assertEquals("Codigo de acesso obrigatorio", resultado.getErrors().get(0))
            );
        }

        @Test
        @DisplayName("Quando alteramos o código de acesso do usuario mais de 6 digitos")
        void quandoAlteramosCodigoAcessoDousuarioMaisDe6Digitos() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setCodigo("1234567");

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isBadRequest())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("Erros de validacao encontrados", resultado.getMessage()),
                    () -> assertEquals("Codigo de acesso deve ter exatamente 6 digitos numericos", resultado.getErrors().get(0))
            );
        }

        @Test
        @DisplayName("Quando alteramos o código de acesso do usuario menos de 6 digitos")
        void quandoAlteramosCodigoAcessoDousuarioMenosDe6Digitos() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setCodigo("12345");

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isBadRequest())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("Erros de validacao encontrados", resultado.getMessage()),
                    () -> assertEquals("Codigo de acesso deve ter exatamente 6 digitos numericos", resultado.getErrors().get(0))
            );
        }

        @Test
        @DisplayName("Quando alteramos o código de acesso do usuario caracteres não numéricos")
        void quandoAlteramosCodigoAcessoDousuarioCaracteresNaoNumericos() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setCodigo("a*c4e@");

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isBadRequest())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("Erros de validacao encontrados", resultado.getMessage()),
                    () -> assertEquals("Codigo de acesso deve ter exatamente 6 digitos numericos", resultado.getErrors().get(0))
            );
        }
    }

    @Nested
    @DisplayName("Conjunto de casos de verificação dos fluxos básicos API Rest")
    class usuarioVerificacaoFluxosBasicosApiRest {

        @Test
        @DisplayName("Quando buscamos por todos usuarios salvos")
        void quandoBuscamosPorTodosusuarioSalvos() throws Exception {
            // Arrange
            // Vamos ter 3 usuarios no banco
            Usuario usuario1 = usuario.builder()
                    .nome("usuario Dois Almeida")
                    .endereco("Av. da Pits A, 100")
                    .codigo("246810")
                    .build();
            Usuario usuario2 = usuario.builder()
                    .nome("usuario Três Lima")
                    .endereco("Distrito dos Testadores, 200")
                    .codigo("135790")
                    .build();
            usuarioRepository.saveAll(Arrays.asList(usuario1, usuario2));

            // Act
            String responseJsonString = driver.perform(get(URI_USUARIOS)
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk()) // Codigo 200
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            List<Usuario> resultado = objectMapper.readValue(responseJsonString, new TypeReference<>() {
            });

            // Assert
            assertAll(
                    () -> assertEquals(3, resultado.size())
            );
        }

        @Test
        @DisplayName("Quando buscamos um usuario salvo pelo id")
        void quandoBuscamosPorUmusuarioSalvo() throws Exception {
            // Arrange
            // nenhuma necessidade além do setup()

            // Act
            String responseJsonString = driver.perform(get(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk()) // Codigo 200
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            UsuarioResponseDTO resultado = objectMapper.readValue(responseJsonString, new TypeReference<>() {});

            // Assert
            assertAll(
                    () -> assertEquals(usuario.getId().longValue(), resultado.getId().longValue()),
                    () -> assertEquals(usuario.getNome(), resultado.getNome())
            );
        }

        @Test
        @DisplayName("Quando buscamos um usuario inexistente")
        void quandoBuscamosPorUmusuarioInexistente() throws Exception {
            // Arrange
            // nenhuma necessidade além do setup()

            // Act
            String responseJsonString = driver.perform(get(URI_USUARIOS + "/" + 999999999)
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isBadRequest()) // Codigo 400
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("O Usuario consultado nao existe!", resultado.getMessage())
            );
        }

        @Test
        @DisplayName("Quando criamos um novo usuario com dados válidos")
        void quandoCriarusuarioValido() throws Exception {
            // Arrange
            // nenhuma necessidade além do setup()

            // Act
            String responseJsonString = driver.perform(post(URI_USUARIOS)
                            .contentType(MediaType.APPLICATION_JSON)
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isCreated()) // Codigo 201
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            Usuario resultado = objectMapper.readValue(responseJsonString, Usuario.UsuarioBuilder.class).build();

            // Assert
            assertAll(
                    () -> assertNotNull(resultado.getId()),
                    () -> assertEquals(usuarioPostPutRequestDTO.getNome(), resultado.getNome())
            );

        }

        @Test
        @DisplayName("Quando alteramos o usuario com dados válidos")
        void quandoAlteramosusuarioValido() throws Exception {
            // Arrange
            Long usuarioId = usuario.getId();

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk()) // Codigo 200
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            Usuario resultado = objectMapper.readValue(responseJsonString, Usuario.UsuarioBuilder.class).build();

            // Assert
            assertAll(
                    () -> assertEquals(resultado.getId().longValue(), usuarioId),
                    () -> assertEquals(usuarioPostPutRequestDTO.getNome(), resultado.getNome())
            );
        }

        @Test
        @DisplayName("Quando alteramos o usuario inexistente")
        void quandoAlteramosusuarioInexistente() throws Exception {
            // Arrange
            // nenhuma necessidade além do setup()

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + 99999L)
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isBadRequest()) // Codigo 400
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("O Usuario consultado nao existe!", resultado.getMessage())
            );
        }

        @Test
        @DisplayName("Quando alteramos o usuario passando código de acesso inválido")
        void quandoAlteramosusuarioCodigoAcessoInvalido() throws Exception {
            // Arrange
            Long usuarioId = usuario.getId();

            // Act
            String responseJsonString = driver.perform(put(URI_USUARIOS + "/" + usuarioId)
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", "invalido")
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isBadRequest()) // Codigo 400
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("Codigo de acesso invalido!", resultado.getMessage())
            );
        }

        @Test
        @DisplayName("Quando excluímos um usuario salvo")
        void quandoExcluimosusuarioValido() throws Exception {
            // Arrange
            // nenhuma necessidade além do setup()

            // Act
            String responseJsonString = driver.perform(delete(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo()))
                    .andExpect(status().isNoContent()) // Codigo 204
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            // Assert
            assertTrue(responseJsonString.isBlank());
        }

        @Test
        @DisplayName("Quando excluímos um usuario inexistente")
        void quandoExcluimosusuarioInexistente() throws Exception {
            // Arrange
            // nenhuma necessidade além do setup()

            // Act
            String responseJsonString = driver.perform(delete(URI_USUARIOS + "/" + 999999)
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo()))
                    .andExpect(status().isBadRequest()) // Codigo 400
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("O Usuario consultado nao existe!", resultado.getMessage())
            );
        }

        @Test
        @DisplayName("Quando excluímos um usuario salvo passando código de acesso inválido")
        void quandoExcluimosusuarioCodigoAcessoInvalido() throws Exception {
            // Arrange
            // nenhuma necessidade além do setup()

            // Act
            String responseJsonString = driver.perform(delete(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", "invalido"))
                    .andExpect(status().isBadRequest()) // Codigo 400
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertAll(
                    () -> assertEquals("Codigo de acesso invalido!", resultado.getMessage())
            );
        }
    }
}


package com.ufcg.psoft.project.controller;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.ufcg.psoft.project.dto.UsuarioResponseDTO;
import com.ufcg.psoft.project.model.Usuario;
import com.ufcg.psoft.project.repository.UsuarioRepository;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

import java.util.ArrayList;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
@DisplayName("Testes do controlador de Usuarios")
public class UsuarioTestAula {

    final String URI_USUARIOS = "/usuario";

    @Autowired
    MockMvc driver;

    @Autowired
    UsuarioRepository usuarioRepository;

    ObjectMapper objectMapper = new ObjectMapper();

    List<UsuarioResponseDTO> usuariosDTO = new ArrayList<>();

    @BeforeEach
    void setup() {
        // Object Mapper suporte para LocalDateTime
        objectMapper.registerModule(new JavaTimeModule());
        Usuario usuario1 = usuarioRepository.save(Usuario.builder()
                .nome("Usuario")
                .endereco("Rua 123")
                .codigo("123456")
                .build()
        );

        Usuario usuario2 = usuarioRepository.save(Usuario.builder()
                .nome("Usuaria")
                .endereco("Rua 234")
                .codigo("123456")
                .build()
        );

        UsuarioResponseDTO r1 = UsuarioResponseDTO.builder()
                .nome(usuario1.getNome())
                .endereco(usuario1.getEndereco())
                .id(usuario1.getId())
                .build();

        usuariosDTO.add(r1);
    }

    @AfterEach
    void tearDown() {
        usuarioRepository.deleteAll();
    }

    @Nested
    @DisplayName("Conjunto de casos de teste da aula")
    class UsuarioVerificacaoNome {

        @Test
        @DisplayName("Quando recuperamos usuarios")
        void quandoRecuperamosUsuariosValidos() throws Exception {

            String stringBusca = "Usuario";
            // Act
            String responseJsonString = driver.perform(get(URI_USUARIOS)
                            .param("nome", stringBusca))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            String expectedResult = objectMapper
                    .writeValueAsString(usuariosDTO);

            System.out.println(expectedResult);
            System.out.println(responseJsonString);
            // Assert
            assertEquals(expectedResult, responseJsonString);
        }

    }
}


package com.ufcg.psoft.project;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

@SpringBootTest
class ProjectApplicationTests {

	@Test
	void contextLoads() {
	}

}
