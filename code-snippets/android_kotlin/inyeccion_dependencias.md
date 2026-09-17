# Inyección de Dependencias con Hilt (Android)

Este fragmento muestra la configuración del módulo principal de **Hilt** para la aplicación Android. Se utiliza para inyectar dependencias como Retrofit, OkHttp, Room Database y los repositorios de datos.

> **Nota de Arquitectura:** Se sigue el patrón **Clean Architecture** con capas separadas (data, domain, presentation). Hilt gestiona el ciclo de vida de las dependencias de forma eficiente.

```kotlin
// di/AppModule.kt

import android.content.Context
import androidx.room.Room
import com.otn.firmaapp.data.api.FdoApiService
import com.otn.firmaapp.data.local.AppDatabase
import com.otn.firmaapp.data.local.dao.DocumentoDao
import com.otn.firmaapp.data.repository.DocumentoRepositoryImpl
import com.otn.firmaapp.domain.repository.DocumentoRepository
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import java.util.concurrent.TimeUnit
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    // URL base del servidor (configurable en tiempo de compilación)
    private const val BASE_URL = "https://tu-sitio-de-pruebas.com/"

    /**
     * Provee una instancia única de OkHttpClient con interceptores para logging y autenticación.
     */
    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        val loggingInterceptor = HttpLoggingInterceptor().apply {
            level = HttpLoggingInterceptor.Level.BODY
        }

        return OkHttpClient.Builder()
            .addInterceptor(loggingInterceptor)
            .addInterceptor { chain ->
                // Añadir token JWT a las peticiones si está disponible
                val request = chain.request().newBuilder()
                    .addHeader("Content-Type", "application/json")
                    .build()
                chain.proceed(request)
            }
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }

    /**
     * Provee la instancia de Retrofit configurada.
     */
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    /**
     * Provee el servicio de API REST.
     */
    @Provides
    @Singleton
    fun provideFdoApiService(retrofit: Retrofit): FdoApiService {
        return retrofit.create(FdoApiService::class.java)
    }

    /**
     * Provee la base de datos Room.
     */
    @Provides
    @Singleton
    fun provideAppDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "fdo_database"
        ).fallbackToDestructiveMigration().build()
    }

    /**
     * Provee el DAO de documentos.
     */
    @Provides
    @Singleton
    fun provideDocumentoDao(database: AppDatabase): DocumentoDao {
        return database.documentoDao()
    }

    /**
     * Provee la implementación del repositorio de documentos.
     */
    @Provides
    @Singleton
    fun provideDocumentoRepository(
        apiService: FdoApiService,
        documentoDao: DocumentoDao
    ): DocumentoRepository {
        return DocumentoRepositoryImpl(apiService, documentoDao)
    }
}
