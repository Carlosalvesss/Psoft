UsuarioRepository


package com.ufcg.psoft.project.repository;

import com.ufcg.psoft.project.model.Usuario;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;

public interface UsuarioRepository extends JpaRepository<Usuario, Long> {

    /*
     * US01 - Buscar usuarios por nome.
     * "Containing"      -> busca parcial (nomes que CONTEM o termo informado).
     * "IgnoreCase"      -> nao diferencia maiusculas de minusculas.
     * "OrderByNomeAsc"  -> US03: resultado ja vem ordenado alfabeticamente pelo nome.
     * Caso nenhum usuario corresponda, o Spring Data retorna uma lista vazia.
     */
    List<Usuario> findByNomeContainingIgnoreCaseOrderByNomeAsc(String nome);

    /*
     * US02 - Buscar usuarios por endereco.
     * Mesma semantica da busca por nome (parcial + ignore case),
     * porem US03 garante que a ordenacao continua sendo pelo NOME,
     * independentemente do criterio de busca utilizado.
     */
    List<Usuario> findByEnderecoContainingIgnoreCaseOrderByNomeAsc(String endereco);
}




UsuarioService


package com.ufcg.psoft.project.service.usuario;

import com.ufcg.psoft.project.dto.UsuarioPostPutRequestDTO;
import com.ufcg.psoft.project.dto.UsuarioResponseDTO;

import java.util.List;

public interface UsuarioService {

    UsuarioResponseDTO alterar(Long id, String codigoAcesso, UsuarioPostPutRequestDTO usuarioPostPutRequestDTO);

    List<UsuarioResponseDTO> listar();

    UsuarioResponseDTO recuperar(Long id);

    UsuarioResponseDTO criar(UsuarioPostPutRequestDTO usuarioPostPutRequestDTO);

    void remover(Long id, String codigoAcesso);

    // US01 - Buscar usuarios por nome (parcial, case-insensitive, ordenado por nome)
    List<UsuarioResponseDTO> buscarPorNome(String nome);

    // US02 - Buscar usuarios por endereco (parcial, case-insensitive, ordenado por nome)
    List<UsuarioResponseDTO> buscarPorEndereco(String endereco);
}




UsuarioServiceImpl



package com.ufcg.psoft.project.service.usuario;

import com.ufcg.psoft.project.exception.UsuarioNaoExisteException;
import com.ufcg.psoft.project.exception.CodigoDeAcessoInvalidoException;
import com.ufcg.psoft.project.model.Usuario;
import com.ufcg.psoft.project.repository.UsuarioRepository;
import com.ufcg.psoft.project.dto.UsuarioPostPutRequestDTO;
import com.ufcg.psoft.project.dto.UsuarioResponseDTO;
import org.modelmapper.ModelMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
public class UsuarioServiceImpl implements UsuarioService {

    @Autowired
    UsuarioRepository usuarioRepository;
    @Autowired
    ModelMapper modelMapper;

    @Override
    public UsuarioResponseDTO alterar(Long id, String codigoAcesso, UsuarioPostPutRequestDTO usuarioPostPutRequestDTO) {
        Usuario usuario = usuarioRepository.findById(id).orElseThrow(UsuarioNaoExisteException::new);
        if (!usuario.getCodigo().equals(codigoAcesso)) {
            throw new CodigoDeAcessoInvalidoException();
        }
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

    // ----------------------------------------------------------------
    // US01 - Buscar usuarios por nome
    // ----------------------------------------------------------------
    @Override
    public List<UsuarioResponseDTO> buscarPorNome(String nome) {
        return usuarioRepository.findByNomeContainingIgnoreCaseOrderByNomeAsc(nome)
                .stream()
                .map(UsuarioResponseDTO::new)
                .collect(Collectors.toList());
    }

    // ----------------------------------------------------------------
    // US02 - Buscar usuarios por endereco
    // US03 - O resultado ja vem ordenado por nome (OrderByNomeAsc),
    //        independentemente de a busca ter sido por nome ou endereco.
    // ----------------------------------------------------------------
    @Override
    public List<UsuarioResponseDTO> buscarPorEndereco(String endereco) {
        return usuarioRepository.findByEnderecoContainingIgnoreCaseOrderByNomeAsc(endereco)
                .stream()
                .map(UsuarioResponseDTO::new)
                .collect(Collectors.toList());
    }
}





UsuarioController



package com.ufcg.psoft.project.controller;

import com.ufcg.psoft.project.dto.UsuarioPostPutRequestDTO;
import com.ufcg.psoft.project.service.usuario.UsuarioService;
import jakarta.validation.Valid;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping(
        value = "/usuario",
        produces = MediaType.APPLICATION_JSON_VALUE
)
public class UsuarioController {

    @Autowired
    UsuarioService usuarioService;

    @GetMapping("/{id}")
    public ResponseEntity<?> recuperarUsuario(
            @PathVariable Long id) {
        return ResponseEntity
                .status(HttpStatus.OK)
                .body(usuarioService.recuperar(id));
    }

    @GetMapping("")
    public ResponseEntity<?> listarUsuarios() {
        return ResponseEntity
                .status(HttpStatus.OK)
                .body(usuarioService.listar());
    }

    @PostMapping()
    public ResponseEntity<?> criarUsuario(
            @RequestBody @Valid UsuarioPostPutRequestDTO usuarioPostPutRequestDto) {
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(usuarioService.criar(usuarioPostPutRequestDto));
    }

    @PutMapping("/{id}")
    public ResponseEntity<?> atualizarUsuario(
            @PathVariable Long id,
            @RequestParam String codigo,
            @RequestBody @Valid UsuarioPostPutRequestDTO usuarioPostPutRequestDto) {
        return ResponseEntity
                .status(HttpStatus.OK)
                .body(usuarioService.alterar(id, codigo, usuarioPostPutRequestDto));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<?> excluirUsuario(
            @PathVariable Long id,
            @RequestParam String codigo) {
        usuarioService.remover(id, codigo);
        return ResponseEntity
                .status(HttpStatus.NO_CONTENT)
                .body("");
    }

    // ----------------------------------------------------------------
    // US01 - Buscar usuarios por nome
    // GET /usuario/busca/nome?nome=...
    // ----------------------------------------------------------------
    @GetMapping("/busca/nome")
    public ResponseEntity<?> buscarUsuariosPorNome(
            @RequestParam String nome) {
        return ResponseEntity
                .status(HttpStatus.OK)
                .body(usuarioService.buscarPorNome(nome));
    }

    // ----------------------------------------------------------------
    // US02 - Buscar usuarios por endereco
    // GET /usuario/busca/endereco?endereco=...
    // ----------------------------------------------------------------
    @GetMapping("/busca/endereco")
    public ResponseEntity<?> buscarUsuariosPorEndereco(
            @RequestParam String endereco) {
        return ResponseEntity
                .status(HttpStatus.OK)
                .body(usuarioService.buscarPorEndereco(endereco));
    }
}







