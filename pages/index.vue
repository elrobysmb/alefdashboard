<template>
  <VContainer fluid class="fill-height">
    <VRow no-gutters align="center" justify="center" class="fill-height">
      <VCol cols="12" md="6" lg="5" sm="6">
        <VRow no-gutters align="center" justify="center">
          <VCol cols="12" md="6">
            <h1>Iniciar Sesión</h1>
            <p class="text-medium-emphasis">Ingresa tus credenciales para iniciar sesión</p>

            <VForm @submit.prevent="submit" class="mt-7">
              <div class="mt-1">
                <label class="label text-grey-darken-2" for="email">Email</label>
                <VTextField :rules="[ruleRequired, ruleEmail]" v-model="email"
                  prepend-inner-icon="fluent:mail-24-regular" id="email" name="email" type="email" />
              </div>
              <div class="mt-1">
                <label class="label text-grey-darken-2" for="password">Contraseña</label>
                <VTextField :rules="[ruleRequired, rulePassLen]" v-model="password"
                  prepend-inner-icon="fluent:password-20-regular" id="password" name="password" type="password" />
              </div>
              <div class="mt-5">
                <VBtn type="submit" block min-height="44" class="gradient primary">Iniciar Sesión</VBtn>
              </div>
            </VForm>
            <p class="text-body-2 mt-10">
              <NuxtLink to="/reset-password" class="font-weight-bold text-primary">¿Olvidaste tu contraseña?</NuxtLink>
            </p>
            <p class="text-body-2 mt-4">
              <span>¿No tienes una cuenta?
                <NuxtLink to="/signup" class="font-weight-bold text-primary">Registrate</NuxtLink>
              </span>
            </p>
          </VCol>
        </VRow>
      </VCol>
      <VCol class="hidden-md-and-down fill-height" md="6" lg="7">
        <div class="h-100 rounded-xl d-flex align-center justify-center" style="position: relative; overflow: hidden;">
          <!-- Blurred background image -->
          <VImg src="@/assets/img/fondo1.jpg" cover class="h-100 rounded-xl"
            style="position: absolute; top: 0; left: 0; width: 100%; height: 100%; filter: blur(5px);" />
          <div class="text-center w-50 text-white mx-auto" style="position: relative; z-index: 1;">
            <h2 class="mb-4">Alef Company</h2>
            <p>
              En ALEF AI, nos dedicamos a transformar la atención al cliente en clínicas mediante RECEPCIONISTAS IA
              avanzados. Nuestro objetivo es automatizar completamente las tareas de una recepcionista, permitiendo que
              las clínicas se enfoquen en brindar atención médica de calidad.
            </p>
          </div>
        </div>
      </VCol>
    </VRow>
  </VContainer>
</template>

<script setup>
import { ref } from 'vue'; // Asegurar importación
const client = useSupabaseClient();
const router = useRouter();

// Estado del formulario
const email = ref(""); // Pre-llenado para pruebas
const password = ref("");
const loading = ref(false);
const errorMsg = ref("");

// Reglas de validación (Simuladas si no tienes el composable importado)
const ruleRequired = (v) => !!v || 'Este campo es requerido';
const ruleEmail = (v) => /.+@.+\..+/.test(v) || 'Email inválido';
const rulePassLen = (v) => (v && v.length >= 6) || 'Mínimo 6 caracteres';

const submit = async () => {
  loading.value = true;
  errorMsg.value = "";

  try {
    let finalSession = null;
    let authSession = null;

    // 1. Intentar Autenticación Nativa (Supabase Auth)
    // Esto es lo ideal para que RLS funcione
    const { data: authData, error: authError } = await client.auth.signInWithPassword({
      email: email.value,
      password: password.value
    });

    if (!authError && authData?.user) {
      console.log("Login nativo exitoso");
      authSession = authData.user;

      // Obtener datos adicionales del perfil si es necesario
      const { data: rpcData } = await client.rpc('login_dashboard', {
        p_email: email.value,
        p_password: password.value
      });

      if (rpcData) {
        finalSession = rpcData;
      } else {
        // Fallback perfil
        const { data: profileData } = await client
          .from('dashboardlogin')
          .select('*')
          .eq('id', authData.user.id)
          .single();

        finalSession = {
          id: authData.user.id,
          email: authData.user.email,
          full_name: profileData?.full_name || profileData?.nombre || 'Usuario Verificado',
          role: profileData?.role || 'authenticated'
        };
      }
    } else {
      console.log("Login nativo falló, intentando RPC legacy...");

      // 2. Si falla Auth nativo, intentar validar con el sistema antiguo (RPC)
      // Esto recupera el acceso al usuario
      const { data: rpcData, error: rpcError } = await client.rpc('login_dashboard', {
        p_email: email.value,
        p_password: password.value
      });

      if (rpcData) {
        console.log("RPC legacy exitoso. Intentando migración automática a Supabase Auth...");
        finalSession = rpcData;

        // 3. MIGRACION AUTOMÁTICA (Auto-SignUp)
        // Como el RPC confirmó que la contraseña es correcta, intentamos crear el usuario en Supabase Auth
        // para que las políticas RLS funcionen en el futuro.
        const { data: signUpData, error: signUpError } = await client.auth.signUp({
          email: email.value,
          password: password.value,
          options: {
            data: {
              full_name: rpcData.nombre || rpcData.full_name // Guardar nombre en metadata
            }
          }
        });

        if (!signUpError && signUpData?.session) {
          console.log("Migración exitosa: Usuario creado y logueado en Auth.");
          // Refrescamos el cliente para tener el token
          await client.auth.setSession(signUpData.session);
        } else if (signUpError) {
          console.warn("Intento de migración falló (posiblemente requiere confirmación de email o ya existe):", signUpError);
        }

      } else {
        throw new Error("Credenciales incorrectas");
      }
    }

    if (finalSession) {
      // Login exitoso
      const userSession = useCookie('dashboard_session');
      userSession.value = finalSession;

      // Redirigir según el correo
      const emailLower = email.value.toLowerCase();

      if (emailLower === "bazanjuanpa@gmail.com") {
        router.push('/pruebas/HealUp');
      } else if (emailLower === "elroby75@hotmail.com") {
        router.push('/pruebas/BradaPerfumes');
      } else {
        router.push('/pruebas/AlefCompany');
      }
    } else {
      alert("No se pudo iniciar sesión. Verifique sus credenciales.");
    }

  } catch (e) {
    console.error("Login Error:", e);
    // Mostrar mensaje amigable
    let msg = "Error de inicio de sesión";
    if (e.message.includes("Invalid login credentials")) msg = "Credenciales inválidas";
    else if (e.message) msg = e.message;

    alert(msg);
  } finally {
    loading.value = false;
  }
};
</script>
