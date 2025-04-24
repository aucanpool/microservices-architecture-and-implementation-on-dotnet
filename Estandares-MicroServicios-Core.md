# Estándares de Desarrollo de Microservicios para el Core Bancario

Este documento establece los estándares para el desarrollo de microservicios en el Core Bancario, asegurando consistencia, calidad y adherencia a principios RestFul.

## Capas de los Microservicios

### 1. Controladores
- **Responsabilidad**: Manejar las solicitudes HTTP y delegar la lógica de negocio a las capas correspondientes.
- **Lógica Permitida**:
    - Validación inicial de parámetros de entrada.
    - Manejo de excepciones y mapeo de errores a respuestas HTTP adecuadas.
    - Transformación de datos de entrada/salida (DTOs), si los DTOs no necesitan mucho tratamiento se puede conservar el DTO y exponerlo en el endpoint.
- **Respuesta**: 
    - Responder con códigos HTTP estándar (200, 201, 400, 404, 500, etc.).
    - Enviar datos en formato JSON o XML según el contrato definido.
  
#### Ejemplo de Controlador:
```java
/**
 * Controller for managing accounts.
 * 
 * @author BRM0000
 * @since 01/01/2025
 */
@Api(tags = "Accounts V1")
@Tag(name = "Accounts V1", description = "Operations related to accounts")
@RestController
@RequestMapping(VERSIONS_ACCOUNTS)
@RequiredArgsConstructor
@ApiResponseAnnotations
public class AccountController {
        private final AccountService accountService;

        /**
         * Handles both blocking and unblocking of accounts.
         * 
         * @param accountModel The model containing account data
         * @return ResponseEntity containing the response model
         * 
         * @author BRM00000
         * @since 01/01/2025
         */
        @Operation(summary = "Update the account status to block or unlock", description = "Blocks or unblocks an account based on the provided information. Allowed status: "
                        + "\"lock\", \"unlock\" ")
        @PostMapping(value = "/{accountNumber}/status", produces = MediaType.APPLICATION_JSON_VALUE)
        public ResponseEntity<AccountStatusDto> updateAccountStatus(
            // @ApiParam nos ayuda a documentar los parametros dentro del endpoint, se recomienda poner un ejemplo del dato
            @PathVariable @ApiParam(value = "The account number you want to update", example = "998170550014", required = true) 
            // @Pattern Nos permite a limitar los datos de entrada con expresiones regulares y dejar un mensaje particular
            @Pattern(regexp = "^\\d{12}$", message = "The account number must be 12 digits") String accountNumber,
            // @Valid nos ayuda a que validar el modelo "AccountStatusModel"
            @Valid @RequestBody AccountStatusModel accountModel) {
                // Respuesta con código 200 y datos en JSON
                return ResponseEntity
                                .ok(accountService.updateAccountStatus(accountNumber,// accountNumber no se sanitiza porque ya tiene filtrado su contenido con @Pattern
                                                Utils.sanitizeObject(accountModel)));// accountModel se sanitiza porque tiene propiedades que son del tipo texto y no estan limitados sus caracteres.
        }

}
```

### 2. Servicios
- **Responsabilidad**: Contener la lógica de negocio principal.
- **Lógica Permitida**:
    - Implementación técnica que coordina y ejecuta las operaciones necesarias para cumplir con las reglas de negocio. La lógica de negocio traduce las reglas de negocio en acciones concretas dentro del sistema.
    - Coordinación entre diferentes componentes del sistema (repositorios, servicios remotos, etc.).
    - Validaciones complejas que no correspondan al controlador.
    - Uso de utilidades comunes para validación de datos especificas de negocio.

#### Ejemplo de Servicio:
```java
@Service
@RequiredArgsConstructor
@Log4j2
public class AccountService {

    private final SibamexAccountRemoteService accountRemoteService;
    private final Validator validator;

    public AccountStatusDto updateAccountStatus(String accountNumber, AccountStatusModel accountModel) {
        if (AccountStatus.UNBLOCK.equals(accountModel.getStatus())) {
            validate(accountModel, UnblockGroup.class, validator);
            return accountRemoteService.unblockAccount(accountNumber);
        } else {
            validate(accountModel, BlockGroup.class, validator);
            return accountRemoteService.blockAccount(accountNumber);
        }
    }
}
```

- **Respuesta**:
    - Retornar objetos de dominio o DTOs procesados.
    - Manejar excepciones internas y propagarlas con mensajes claros.

#### Ejemplo de Manejo de Excepciones:
```java
/**
     * Blocks an account based on the provided account number and account model.
     * This method interacts with the remote account service to block the account
     * and validates the response to ensure a valid block number is generated.
     *
     * @param accountNumber the account number to be blocked
     * @param accountModel  the account model containing additional block details
     * @return an AccountStatusDto containing the details of the blocked account
     * @throws KrAccountException if the response is empty, or the generated block
     *                            number
     *                            is null or less than or equal to zero
     */
    private AccountStatusDto blockAccount(String accountNumber, AccountStatusModel accountModel) {
        AccountStatusDto accountStatusDto = accountRemoteService
                .blockAccount(AccountFactory.createAccountBlockDto(accountNumber, accountModel));

        if (ObjectUtils.isEmpty(accountStatusDto) ||
                accountStatusDto.getGeneratedBlockNumber() == null ||
                accountStatusDto.getGeneratedBlockNumber() <= 0) {
            throw new KrAccountException(KrAccountError.BLOCKED_ACCOUNT_NO_BLOCK_NUMBER,
                    HttpStatus.INTERNAL_SERVER_ERROR);
        }
        return accountStatusDto;
    }
```

### 3. Servicios Remotos
- **Responsabilidad**: Interactuar con servicios externos utilizando patrones de resiliencia.
- **Lógica Permitida**:
    - Implementación del patrón Circuit Breaker para garantizar resiliencia.
    - Manejo de tiempos de espera y reintentos controlados.
    - Transformación de respuestas externas a objetos internos.

#### Ejemplo de Servicio Remoto con Circuit Breaker:
```java
@Log4j2
@Service
@RequiredArgsConstructor
public class SibamexAccountRemoteService {

  private final SibamexAccountRemoteRepository sibamexAccountRemoteRepository;
  private static final String REMOTE_CODE = "ECSICU";

  /**
   * Executes a remote account blocking operation via SIBAMEX service with circuit
   * breaker and retry protection.
   */
  @CircuitBreaker(name = "sibamexService", fallbackMethod = "fallbackBlockAccount")
  @Retry(name = "sibamexServiceRetry")
  public AccountStatusDto blockAccount(AccountBlockDto accountBlockDto) {
    return sibamexAccountRemoteRepository.blockAccount(accountBlockDto);
  }

  /**
   * Fallback method for blockAccount in case of failure.
   *
   * @param accountBlockDto The DTO containing the account information to be
   *                        blocked.
   * @param throwable       The exception that caused the fallback.
   * @return AccountStatusDto with a failure status.
   */
  public AccountStatusDto fallbackBlockAccount(AccountBlockDto accountBlockDto, Throwable throwable) {
    log.error("Fallback triggered for blockAccount: {}", throwable.toString());
    return AccountFactory.createAccountBlockDto(Constants.ZERO);
  }
```

- **Respuesta**:
    - Retornar datos procesados desde servicios externos.
    - Manejar errores externos y mapearlos a excepciones internas.

### 4. Repositorios Remotos
- **Responsabilidad**: Realizar llamadas HTTP a servicios externos utilizando FeignClient.
- **Lógica Permitida**:
    - Configuración de endpoints y mapeo de solicitudes/respuestas.
    - Manejo de autenticación y encabezados necesarios.

#### Ejemplo de Repositorio Remoto con FeignClient:
```java
@FeignClient("${sibamex-cuenta.uri}")
public interface SibamexAccountRemoteRepository {

  /**
   * Makes a request to block account.
   *
   * @param accountBlock The account block information contained in a DTO
   * @return AccountBlockResponseDto with the response of the block operation
   */
  @PostMapping("/v1/cuenta/bloqueos")
  public AccountStatusDto blockAccount(@RequestBody AccountBlockDto accountBlockDto);

  /**
   * Unblocks the account through the remote service.
   *
   * @param accountLockModel The DTO containing the account unblock information
   * @return AccountUnblockResponseDto The response containing the result of the
   *         unblock operation
   * @throws RemoteServiceException if there is an error communicating with the
   *                                remote service
   */
  @PostMapping("/v1/cuenta/desbloqueos")
  public AccountStatusDto unblockAccount(@RequestBody AccountUnblockDto accountUnblockDto);
}
```

- **Respuesta**:
    - Retornar datos crudos desde los servicios externos.
    - Manejar errores HTTP y propagarlos adecuadamente.

## Consideraciones Generales
- **Manejo de Excepciones**: Todas las capas deben manejar excepciones de manera clara y consistente, propagando mensajes útiles para el diagnóstico.
Ejemplo de Manejo de Excepciones Global:
```java
@ControllerAdvice
@Log4j2
public class GlobalExceptionHandler {
    public GlobalExceptionHandler(ErrorAttributes errorAttributes) {
        this.errorAttributes = errorAttributes;
    }

    // Handle errors when JSON is not well-formed (invalid syntax)
    @ExceptionHandler(JsonParseException.class)
    public ResponseEntity<Map<String, Object>> handleJsonParseException(JsonParseException ex, WebRequest webRequest) {
        KrAccountException krAccountException = new KrAccountException(
                KrAccountError.BAD_REQUEST,
                HttpStatus.BAD_REQUEST, JSON_PARSE_EXCEPTION_MESSAGE);
        return getNextErrorAttributes(webRequest, krAccountException, ErrorAttributeOptions.defaults());
    }

    private ResponseEntity<Map<String, Object>> getNextErrorAttributes(WebRequest webRequest,
            KrAccountException krAccountException, ErrorAttributeOptions options) {
        webRequest.setAttribute(ORG_SPRINGFRAMEWORK_BOOT_WEB_SERVLET_ERROR_DEFAULT_ERROR_ATTRIBUTES_ERROR,
                krAccountException, RequestAttributes.SCOPE_REQUEST);
        String requestUri = DIAGONAL_STRING;
        if (webRequest instanceof ServletWebRequest) {
            requestUri = ((ServletWebRequest) webRequest).getRequest().getRequestURI();
        }
        webRequest.setAttribute(JAVAX_SERVLET_ERROR_REQUEST_URI,
                requestUri, RequestAttributes.SCOPE_REQUEST);
        long duration = BaseUtil.extractDurationRequest(webRequest);
        MDC.put(BaseConstantes.MDC_DURATION_KEY, String.valueOf(duration));
        options = options.including(Include.MESSAGE);
        Map<String, Object> errorAttributesMap = errorAttributes.getErrorAttributes(webRequest,
                options);
        return new ResponseEntity<>(errorAttributesMap, krAccountException.getStatus());
    }

    // Helper method to get field name in InvalidFormatException

    private String extractFieldName(InvalidFormatException ex) {
        if (!ex.getPath().isEmpty()) {
            // gets the last element of the field path, which corresponds to the
            // field name
            return ex.getPath().get(ex.getPath().size() - 1).getFieldName();
        }
        return UNKNOWN_FIELD;
    }
}
```
- **Pruebas Unitarias**: Cada capa debe contar con pruebas unitarias que validen su funcionalidad de forma aislada.
Ejemplo de Prueba Unitaria para Servicio:
```java
@Test
public void testGetAccountById() {
    Long accountId = 1L;
    AccountDTO mockAccount = new AccountDTO(accountId, "Cuenta de prueba");
    when(accountRepository.findById(accountId)).thenReturn(Optional.of(mockAccount));

    AccountDTO result = accountService.getAccountById(accountId);

    assertEquals(mockAccount, result);
    verify(accountRepository, times(1)).findById(accountId);
}
```
- **Documentación**: Los servicios deben estar documentados utilizando herramientas como Swagger para facilitar su consumo.
Ejemplo de Documentación con Swagger:
```java
@ApiOperation(value = "Obtener cuenta por ID", notes = "Devuelve los detalles de una cuenta bancaria específica.")
@ApiResponses(value = {
    @ApiResponse(code = 200, message = "Cuenta obtenida exitosamente."),
    @ApiResponse(code = 404, message = "Cuenta no encontrada.")
})
@GetMapping("/{id}")
public ResponseEntity<AccountDTO> getAccountById(@PathVariable Long id) {
    return ResponseEntity.ok(accountService.getAccountById(id));
}
```
- **Excepciones personalizadas del Microsservicio** : Los micro servicios deben tener su propia excepción y toda excepción controlada en este ms debe generarse con la misma y debe extender de NextAbtractException.
Ejemplo de excepción personalizada del micro servicio.
```java
public class KrAccountException extends NextAbtractException {

    public KrAccountException(NextException code) {
        super(code, code.getMessage(), (Throwable) null, HttpStatus.CONFLICT);
    }

    public KrAccountException(NextException code, HttpStatus status) {
        super(code, code.getMessage(), (Throwable) null, status);
    }

    public KrAccountException(String detailMessage, NextException code, HttpStatus httpStatus) {
        super(code, detailMessage, null, httpStatus);
    }

    public KrAccountException(NextException code, HttpStatus httpStatus, String detailMessage,
            Object... parameters) {
        super(code, String.format(detailMessage, parameters), null, httpStatus);
    }

    public KrAccountException(NextException code, HttpStatus httpStatus, Throwable cause, String detailMessage,
            Object... parameters) {
        super(code, String.format(detailMessage, parameters), cause, httpStatus);
    }
}
```
- **Enumerador de errores personalizadas del Microsservicio** : Los micro servicios deben de tener enumerador de los errores que se generan con las excepciones
```java
public enum KrAccountError implements NextException {
	BAD_REQUEST("Bad request"),
	FAILED_PROCESS("Could not block the account. Please verify the data and try again. %s"),
	ERROR_PROCESS("Error processing account unblock for account %s"),
	ACCOUNT_NOT_FOUND("Account not found"),
	ACCOUNT_HAS_BALANCE("Account has balance and cannot be canceled"),
	CANCELLATION_FAILED("Cancellation failed due to business rules"),
	INVALID_INPUT("Invalid input for account operation");

	private final String message;

	private KrAccountError(String message) {
		this.message = message;
	}

	@Override
	public String getMessage() {
		return message;
	}
}
```