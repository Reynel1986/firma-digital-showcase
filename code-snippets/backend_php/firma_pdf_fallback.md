# Firma Digital de PDFs (OpenSSL + TCPDF + FPDI)

Este fragmento muestra el núcleo del sistema de firma. El proceso consta de tres pasos:
1. Leer el certificado `.p12` y extraer la clave privada.
2. Convertir el PDF original con **Ghostscript** para asegurar compatibilidad con FPDI (evitando errores de compresión avanzada).
3. Aplicar la firma criptográfica y visual usando **TCPDF** y **FPDI**.

> **Nota Técnica:** Se utiliza una estrategia de fallback para garantizar que la mayoría de los PDFs puedan ser firmados, incluso si utilizan compresión avanzada (Object Streams).

```php
<?php
// public/class-firma-handler.php

class FDO_Firma_Handler {

    /**
     * Firma un PDF usando OpenSSL y TCPDF.
     */
    private static function firmar_con_openssl_tcpdf($pdf_content, $pfx_content, $password, $coordenadas = null, $ruta_imagen_firma = null) {
        try {
            // 1. Extraer clave privada y certificado del PFX
            if (!openssl_pkcs12_read($pfx_content, $cert_info, $password)) {
                $error = openssl_error_string();
                throw new Exception("Error al leer el certificado: $error");
            }
            
            $cert_pem = $cert_info['cert'];
            $private_key = $cert_info['pkey'];
            
            // 2. Crear archivos temporales para el PDF
            $temp_pdf = tempnam(sys_get_temp_dir(), 'fdo_pdf_');
            file_put_contents($temp_pdf, $pdf_content);
            
            // 3. Convertir con Ghostscript para compatibilidad con FPDI
            $temp_converted = tempnam(sys_get_temp_dir(), 'fdo_conv_') . '.pdf';
            $command = "gs -dNOPAUSE -dBATCH -sDEVICE=pdfwrite -dPDFSETTINGS=/prepress -sOutputFile=$temp_converted $temp_pdf 2>&1";
            exec($command, $output, $returnCode);
            
            $pdf_to_use = $temp_pdf;
            if ($returnCode === 0 && file_exists($temp_converted) && filesize($temp_converted) > 1000) {
                $pdf_to_use = $temp_converted;
            }
            
            // 4. Cargar dependencias de Composer
            $autoload_path = dirname(__DIR__, 1) . '/vendor/autoload.php';
            if (file_exists($autoload_path)) {
                require_once($autoload_path);
            }
            
            $pdf_firmado = null;
            
            // 5. Firmar usando FPDI y TCPDF
            if (class_exists('setasign\Fpdi\Fpdi')) {
                $fpdi = new \setasign\Fpdi\Fpdi();
                $pageCount = $fpdi->setSourceFile($pdf_to_use);
                
                $tcpdf_path = dirname(__DIR__, 1) . '/vendor/tecnickcom/tcpdf/tcpdf.php';
                if (file_exists($tcpdf_path)) {
                    require_once($tcpdf_path);
                }
                
                $pdf = new TCPDF(PDF_PAGE_ORIENTATION, PDF_UNIT, PDF_PAGE_FORMAT, true, 'UTF-8', false);
                $pdf->setPrintHeader(false);
                $pdf->setPrintFooter(false);
                $pdf->SetMargins(0, 0, 0);
                $pdf->SetAutoPageBreak(false, 0);
                
                // Importar páginas del PDF original
                for ($i = 1; $i <= $pageCount; $i++) {
                    $templateId = $fpdi->importPage($i);
                    $pdf->AddPage();
                    $pdf->useTemplate($templateId);
                }
                
                // Insertar imagen de firma si se proporcionó
                if ($ruta_imagen_firma && file_exists($ruta_imagen_firma)) {
                    $pagina_firma = $coordenadas['pagina'] ?? $pageCount;
                    $pdf->setPage($pagina_firma);
                    $pdf->Image($ruta_imagen_firma, $coordenadas['x'] ?? 10, $coordenadas['y'] ?? 10, $coordenadas['ancho'] ?? 80, $coordenadas['alto'] ?? 30, 'PNG');
                }
                
                // Configurar firma criptográfica
                $cert_info_array = openssl_x509_parse($cert_pem);
                $info = [
                    'Name' => $cert_info_array['subject']['CN'] ?? 'Firmante Digital',
                    'Location' => 'Firma Digital OTN',
                    'Reason' => 'Aprobación de documento',
                    'ContactInfo' => $cert_info_array['subject']['emailAddress'] ?? ''
                ];
                
                $pdf->setSignature($cert_pem, $private_key, '', '', 2, $info);
                $pdf_firmado = $pdf->Output('', 'S');
            }
            
            // 6. Limpiar archivos temporales
            self::limpiar_archivos_temporales([$temp_pdf, $temp_converted]);
            
            unset($cert_pem);
            unset($private_key);
            unset($cert_info);
            
            if ($pdf_firmado === null) {
                throw new Exception('No se pudo firmar el documento. El formato del PDF no es compatible con FPDI gratuito.');
            }
            
            return $pdf_firmado;
            
        } catch (Exception $e) {
            error_log('FDO: Excepción al firmar PDF: ' . $e->getMessage());
            throw $e;
        }
    }
}
