# Lógica de Firma de PDF en Android (Kotlin + iText 7)

Este fragmento muestra cómo se aplica una firma digital a un PDF dentro de la aplicación Android. Se utiliza la librería **iText 7** para la manipulación criptográfica del documento.

> **Nota de Seguridad:** El certificado y la contraseña se manejan en memoria y nunca se persisten en el dispositivo. La firma se aplica tras la autenticación biométrica del usuario.

```kotlin
// utils/PdfSignerHelper.kt

import com.itextpdf.signatures.*
import com.itextpdf.kernel.pdf.*
import java.io.InputStream
import java.io.OutputStream
import java.security.KeyStore
import java.security.PrivateKey
import java.security.cert.Certificate

object PdfSignerHelper {

    /**
     * Firma un PDF usando un certificado .p12 cargado desde el sistema o desde un archivo.
     *
     * @param inputStream Flujo del PDF original.
     * @param outputStream Flujo donde se escribirá el PDF firmado.
     * @param p12InputStream Flujo del certificado .p12.
     * @param password Contraseña del certificado.
     * @param reason Razón de la firma (ej. "Aprobación de documento").
     * @param location Ubicación (ej. "Firma Digital OTN").
     */
    fun signPdf(
        inputStream: InputStream,
        outputStream: OutputStream,
        p12InputStream: InputStream,
        password: CharArray,
        reason: String = "Aprobación de documento",
        location: String = "Firma Digital OTN"
    ) {
        // 1. Cargar el certificado .p12 en memoria
        val keyStore = KeyStore.getInstance("PKCS12")
        keyStore.load(p12InputStream, password)

        val alias = keyStore.aliases().nextElement()
        val privateKey: PrivateKey = keyStore.getKey(alias, password) as PrivateKey
        val certificateChain: Array<Certificate> = keyStore.getCertificateChain(alias)

        // 2. Configurar el firmador de iText 7
        val reader = PdfReader(inputStream)
        val writer = PdfWriter(outputStream)
        val pdfDoc = PdfDocument(reader, writer)

        val signer = PrivateKeySignature(privateKey, DigestAlgorithms.SHA256, "BC")
        val stamper = PdfSignatureAppearance(pdfDoc, "FirmaDigital", 
            PdfSignatureAppearance.WHY, 
            PdfSignatureAppearance.LOCATION)

        // 3. Crear la apariencia visual de la firma (opcional)
        stamper.setReason(reason)
        stamper.setLocation(location)
        stamper.setSignatureCreator("Firma Digital OTN")
        stamper.setPageNumber(1)
        stamper.setRectangle(com.itextpdf.kernel.geom.Rectangle(50f, 50f, 200f, 100f))

        // 4. Aplicar la firma criptográfica
        val dic = stamper.getSignatureDictionary()
        dic.put(PdfName.ContactInfo, PdfString(certificateChain[0].toString()))

        val signature = PdfSignerUtils.createSignature(signer, certificateChain, "BC")
        stamper.setSignature(signature)
        stamper.signDetached(signer, certificateChain, null, null, null, 0, PdfSignatureAppearance.CryptoStandard.CMS)

        // 5. Cerrar recursos
        pdfDoc.close()
    }
}
