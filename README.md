HistoricoControllerTests

package com.ufcg.psoft.project.controller;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.ufcg.psoft.project.dto.HistoricoAlteracaoResponseDTO;
import com.ufcg.psoft.project.dto.UsuarioPostPutRequestDTO;
import com.ufcg.psoft.project.exception.CustomErrorType;
import com.ufcg.psoft.project.model.Usuario;
import com.ufcg.psoft.project.repository.HistoricoAlteracaoRepository;
import com.ufcg.psoft.project.repository.UsuarioRepository;
import org.junit.jupiter.api.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.util.List;

import static org.junit.jupiter.api.Assertions.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.put;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@SpringBootTest
@AutoConfigureMockMvc
@DisplayName("Testes do controlador de Historico")
public class HistoricoControllerTests {

    final String URI_HISTORICO = "/historico";
    final String URI_USUARIOS = "/usuario";

    @Autowired
    MockMvc driver;

    @Autowired
    UsuarioRepository usuarioRepository;

    @Autowired
    HistoricoAlteracaoRepository historicoRepository;

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
        historicoRepository.deleteAll();
        usuarioRepository.deleteAll();
    }

    @Nested
    @DisplayName("US01 - Registrar historico de alteracoes")
    class RegistrarHistorico {

        @Test
        @DisplayName("Quando alteramos o endereco, um registro de historico e criado")
        void quandoAlteramosEnderecoCriaHistorico() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setEndereco("Endereco Alterado");

            // Act
            driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk())
                    .andDo(print());

            String responseJsonString = driver.perform(get(URI_HISTORICO + "/usuario/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            List<HistoricoAlteracaoResponseDTO> resultado = objectMapper.readValue(responseJsonString, new TypeReference<>() {});

            // Assert
            assertAll(
                    () -> assertEquals(1, resultado.size()),
                    () -> assertEquals(usuario.getId().longValue(), resultado.get(0).getUsuarioId().longValue()),
                    () -> assertEquals("endereco", resultado.get(0).getCampoAlterado()),
                    () -> assertEquals("Rua dos Testes, 123", resultado.get(0).getValorAnterior()),
                    () -> assertEquals("Endereco Alterado", resultado.get(0).getValorNovo()),
                    () -> assertNotNull(resultado.get(0).getDataHora())
            );
        }

        @Test
        @DisplayName("Quando alteramos o codigo, um registro de historico e criado")
        void quandoAlteramosCodigoCriaHistorico() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setCodigo("654321");

            // Act
            driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk())
                    .andDo(print());

            String responseJsonString = driver.perform(get(URI_HISTORICO + "/usuario/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            List<HistoricoAlteracaoResponseDTO> resultado = objectMapper.readValue(responseJsonString, new TypeReference<>() {});

            // Assert
            assertAll(
                    () -> assertEquals(1, resultado.size()),
                    () -> assertEquals("codigo", resultado.get(0).getCampoAlterado()),
                    () -> assertEquals("123456", resultado.get(0).getValorAnterior()),
                    () -> assertEquals("654321", resultado.get(0).getValorNovo())
            );
        }

        @Test
        @DisplayName("Quando alteramos codigo e endereco, dois registros sao criados")
        void quandoAlteramosCodigoEEnderecoCriaDoisHistoricos() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setEndereco("Endereco Alterado");
            usuarioPostPutRequestDTO.setCodigo("654321");

            // Act
            driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk())
                    .andDo(print());

            String responseJsonString = driver.perform(get(URI_HISTORICO + "/usuario/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            List<HistoricoAlteracaoResponseDTO> resultado = objectMapper.readValue(responseJsonString, new TypeReference<>() {});

            // Assert
            assertEquals(2, resultado.size());
        }

        @Test
        @DisplayName("Quando os dados informados sao identicos, nenhum registro e criado")
        void quandoDadosIdenticosNaoCriaHistorico() throws Exception {
            // Arrange
            // usuarioPostPutRequestDTO ja possui os mesmos dados do usuario salvo

            // Act
            driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk())
                    .andDo(print());

            String responseJsonString = driver.perform(get(URI_HISTORICO + "/usuario/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            List<HistoricoAlteracaoResponseDTO> resultado = objectMapper.readValue(responseJsonString, new TypeReference<>() {});

            // Assert
            assertTrue(resultado.isEmpty());
        }

        @Test
        @DisplayName("Quando alteramos apenas o nome, nenhum registro e criado")
        void quandoAlteramosApenasNomeNaoCriaHistorico() throws Exception {
            // Arrange
            usuarioPostPutRequestDTO.setNome("Nome Alterado");

            // Act
            driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", usuario.getCodigo())
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk())
                    .andDo(print());

            String responseJsonString = driver.perform(get(URI_HISTORICO + "/usuario/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            List<HistoricoAlteracaoResponseDTO> resultado = objectMapper.readValue(responseJsonString, new TypeReference<>() {});

            // Assert
            assertTrue(resultado.isEmpty());
        }
    }

    @Nested
    @DisplayName("US02 - Consultar historico de um usuario")
    class HistoricoDeUmUsuario {

        @Test
        @DisplayName("Quando o usuario nao possui alteracoes, uma lista vazia e retornada")
        void quandoUsuarioSemAlteracoesRetornaListaVazia() throws Exception {
            // Arrange
            // nenhuma necessidade alem do setup()

            // Act
            String responseJsonString = driver.perform(get(URI_HISTORICO + "/usuario/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            List<HistoricoAlteracaoResponseDTO> resultado = objectMapper.readValue(responseJsonString, new TypeReference<>() {});

            // Assert
            assertTrue(resultado.isEmpty());
        }

        @Test
        @DisplayName("Quando consultamos, apenas os registros do usuario informado sao retornados, em ordem cronologica")
        void quandoConsultamosRetornaApenasDoUsuarioEmOrdem() throws Exception {
            // Arrange
            Usuario outro = usuarioRepository.save(Usuario.builder()
                    .nome("usuario Dois")
                    .endereco("Rua C, 300")
                    .codigo("000000")
                    .build()
            );

            // muda o endereco do usuario
            usuarioPostPutRequestDTO.setEndereco("Endereco Alterado");
            driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", "123456")
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk());

            // muda o codigo do usuario (endereco permanece igual ao atual, nao gera registro)
            usuarioPostPutRequestDTO.setCodigo("111111");
            driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", "123456")
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk());

            // altera o outro usuario (nao deve aparecer no resultado)
            UsuarioPostPutRequestDTO dtoOutro = UsuarioPostPutRequestDTO.builder()
                    .nome("usuario Dois")
                    .endereco("Rua D, 400")
                    .codigo("000000")
                    .build();
            driver.perform(put(URI_USUARIOS + "/" + outro.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", "000000")
                            .content(objectMapper.writeValueAsString(dtoOutro)))
                    .andExpect(status().isOk());

            // Act
            String responseJsonString = driver.perform(get(URI_HISTORICO + "/usuario/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            List<HistoricoAlteracaoResponseDTO> resultado = objectMapper.readValue(responseJsonString, new TypeReference<>() {});

            // Assert
            assertAll(
                    () -> assertEquals(2, resultado.size()),
                    () -> assertTrue(resultado.stream().allMatch(h -> h.getUsuarioId().equals(usuario.getId()))),
                    () -> assertEquals("endereco", resultado.get(0).getCampoAlterado()),
                    () -> assertEquals("codigo", resultado.get(1).getCampoAlterado())
            );
        }

        @Test
        @DisplayName("Quando consultamos o historico de um usuario inexistente")
        void quandoConsultamosHistoricoDeUsuarioInexistente() throws Exception {
            // Arrange
            // nenhuma necessidade alem do setup()

            // Act
            String responseJsonString = driver.perform(get(URI_HISTORICO + "/usuario/" + 999999)
                            .contentType(MediaType.APPLICATION_JSON))
                    .andExpect(status().isBadRequest())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            CustomErrorType resultado = objectMapper.readValue(responseJsonString, CustomErrorType.class);

            // Assert
            assertEquals("O Usuario consultado nao existe!", resultado.getMessage());
        }
    }

    @Nested
    @DisplayName("US03 - Consultar historico geral de alteracoes")
    class HistoricoGeral {

        @Test
        @DisplayName("Quando consultamos, os registros de todos os usuarios sao retornados")
        void quandoConsultamosRetornaTodos() throws Exception {
            // Arrange
            Usuario outro = usuarioRepository.save(Usuario.builder()
                    .nome("usuario Dois")
                    .endereco("Rua C, 300")
                    .codigo("000000")
                    .build()
            );

            usuarioPostPutRequestDTO.setEndereco("Endereco Alterado");
            driver.perform(put(URI_USUARIOS + "/" + usuario.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", "123456")
                            .content(objectMapper.writeValueAsString(usuarioPostPutRequestDTO)))
                    .andExpect(status().isOk());

            UsuarioPostPutRequestDTO dtoOutro = UsuarioPostPutRequestDTO.builder()
                    .nome("usuario Dois")
                    .endereco("Rua D, 400")
                    .codigo("000000")
                    .build();
            driver.perform(put(URI_USUARIOS + "/" + outro.getId())
                            .contentType(MediaType.APPLICATION_JSON)
                            .param("codigo", "000000")
                            .content(objectMapper.writeValueAsString(dtoOutro)))
                    .andExpect(status().isOk());

            // Act
            String responseJsonString = driver.perform(get(URI_HISTORICO)
                            .contentType(MediaType.APPLICATION_JSON))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            List<HistoricoAlteracaoResponseDTO> resultado = objectMapper.readValue(responseJsonString, new TypeReference<>() {});

            // Assert
            assertEquals(2, resultado.size());
        }

        @Test
        @DisplayName("Quando nao existem alteracoes, uma lista vazia e retornada")
        void quandoNaoExistemAlteracoesRetornaListaVazia() throws Exception {
            // Arrange
            // nenhuma necessidade alem do setup()

            // Act
            String responseJsonString = driver.perform(get(URI_HISTORICO)
                            .contentType(MediaType.APPLICATION_JSON))
                    .andExpect(status().isOk())
                    .andDo(print())
                    .andReturn().getResponse().getContentAsString();

            List<HistoricoAlteracaoResponseDTO> resultado = objectMapper.readValue(responseJsonString, new TypeReference<>() {});

            // Assert
            assertTrue(resultado.isEmpty());
        }
    }
}