# Registro de API REST y Validación JWT (WordPress)

Este fragmento muestra cómo se registran los endpoints de la API REST que consume la App Android. Se utiliza la función `register_rest_route` de WordPress, asegurando que solo usuarios autenticados con permisos específicos puedan acceder.

> **Nota de Seguridad:** Este código ha sido sanitizado. No contiene URLs reales, nombres de servidores, credenciales ni tokens expuestos.

```php
<?php
// includes/class-rest-api.php

// Prevenir acceso directo
if (!defined('ABSPATH')) {
    exit;
}

class FDO_REST_API {

    public function __construct() {
        add_action('rest_api_init', array($this, 'registrar_rutas'));
    }

    public function registrar_rutas() {
        // Endpoint para obtener los documentos del usuario
        register_rest_route('fdo/v1', '/documentos', array(
            'methods'  => 'GET',
            'callback' => array($this, 'get_user_documents'),
            'permission_callback' => array($this, 'check_jwt_permission'), // Validación de seguridad
        ));

        // Endpoint para subir un PDF firmado
        register_rest_route('fdo/v1', '/documentos/(?P<id>\d+)/upload-firmado', array(
            'methods'  => 'POST',
            'callback' => array($this, 'upload_signed_pdf'),
            'permission_callback' => array($this, 'check_jwt_permission'),
            'args' => array(
                'id' => array(
                    'validate_callback' => function($param) { return is_numeric($param); }
                ),
            ),
        ));
    }

    /**
     * Verifica que el usuario tenga un token JWT válido y los permisos necesarios.
     */
    public function check_jwt_permission($request) {
        // Lógica de validación de JWT (simplificada para el ejemplo)
        $token = $request->get_header('Authorization');
        
        if (!$token) {
            return new WP_Error('unauthorized', 'Token no proporcionado', array('status' => 401));
        }
        
        // Verificar validez del token con el plugin JWT Authentication for WP REST API
        // ...
        
        return true; 
    }
}
