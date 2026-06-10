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