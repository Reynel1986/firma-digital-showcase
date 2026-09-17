# Cifrado y Almacenamiento Seguro de Certificados Digitales (WordPress)

Este fragmento muestra cómo se gestiona el almacenamiento seguro de los certificados digitales (.p12) de los usuarios. Los certificados nunca se guardan en texto plano en la base de datos; se cifran utilizando **AES-256-CBC** con una clave generada automáticamente por WordPress.

> **Nota de Seguridad:** La clave de cifrado se guarda en las opciones de WordPress. El certificado en sí nunca se expone en los logs ni en la base de datos sin cifrar.

```php
<?php
// includes/class-certificado-usuario.php

class FDO_Certificado_Usuario {

    /**
     * Encriptar certificado usando OpenSSL (AES-256-CBC)
     */
    private static function encriptar_certificado($contenido) {
        // Obtener o generar la clave de encriptación
        $key = get_option('fdo_encryption_key');
        if (!$key) {
            $key = wp_generate_password(32, true, true);
            update_option('fdo_encryption_key', $key);
        }

        // Generar un Vector de Inicialización (IV) seguro
        $iv_length = openssl_cipher_iv_length('aes-256-cbc');
        $iv = openssl_random_pseudo_bytes($iv_length);

        // Encriptar el contenido
        $encriptado = openssl_encrypt($contenido, 'aes-256-cbc', $key, 0, $iv);
        
        if ($encriptado === false) {
            error_log('FDO: Error al encriptar el certificado.');
            return false;
        }

        // Concatenar IV + Contenido Encriptado y codificar en Base64 para guardar en BD
        return base64_encode($iv . $encriptado);
    }

    /**
     * Desencriptar certificado
     */
    private static function desencriptar_certificado($contenido_encriptado) {
        if (empty($contenido_encriptado)) {
            return false;
        }

        $key = get_option('fdo_encryption_key');
        if (!$key) {
            error_log('FDO: No existe clave de encriptación para desencriptar.');
            return false;
        }

        // Decodificar Base64
        $contenido = base64_decode($contenido_encriptado);
        if ($contenido === false) {
            return false;
        }

        $iv_length = openssl_cipher_iv_length('aes-256-cbc');
        
        // Verificar que el contenido tenga la longitud mínima para extraer el IV
        if (strlen($contenido) < $iv_length) {
            return false;
        }

        // Extraer IV y el contenido encriptado
        $iv = substr($contenido, 0, $iv_length);
        $encriptado = substr($contenido, $iv_length);

        // Desencriptar
        $resultado = openssl_decrypt($encriptado, 'aes-256-cbc', $key, 0, $iv);
        
        if ($resultado === false) {
            error_log('FDO: Error al desencriptar el certificado (posible clave incorrecta).');
            return false;
        }

        return $resultado;
    }
}
