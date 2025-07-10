Resolución de Referencias Circulares en TypeORM con ESM
Este documento describe una solución para manejar referencias circulares en TypeORM cuando se utilizan Módulos ECMAScript (ESM) en un proyecto TypeScript. Se aborda el desafío de definir relaciones bidireccionales entre entidades (por ejemplo, User y Reputation) sin causar problemas de importaciones circulares, manteniendo la seguridad de tipos y evitando dependencias globales como globalThis o require de CommonJS.
Problema
En un proyecto TypeORM que utiliza ESM, las relaciones bidireccionales entre entidades pueden provocar importaciones circulares. Por ejemplo:

La entidad User tiene una relación @OneToOne con Reputation, lo que requiere importar Reputation.
La entidad Reputation tiene una relación @OneToOne con User, lo que requiere importar User.
Esto crea una dependencia circular: User -> Reputation -> User.

Las importaciones directas en ESM causan errores porque las importaciones estáticas se resuelven antes de la ejecución, lo que lleva a referencias indefinidas. Soluciones comunes como require (de CommonJS) no son compatibles con ESM, y usar globalThis con any compromete la seguridad de tipos.
Solución
La solución recomendada es usar nombres de entidades (cadenas de texto) en los decoradores de relaciones de TypeORM en lugar de importar directamente las clases de las entidades. Esto evita las importaciones estáticas, rompiendo la dependencia circular. Se utilizan interfaces de TypeScript para mantener la seguridad de tipos sin necesidad de importaciones.
Implementación
1. Definir Entidades con Nombres de Entidades en las Relaciones
En lugar de importar la clase de la entidad relacionada, usa el nombre de la entidad como una cadena en los decoradores @OneToOne, @OneToMany, u otros decoradores de relaciones. TypeORM resuelve estos nombres en tiempo de ejecución, siempre que las entidades estén registradas en el DataSource.
Ejemplo para la entidad User:
// src/models/user.ts
import { PrimaryGeneratedColumn, Column, BaseEntity, OneToMany, OneToOne, JoinColumn, Entity } from 'typeorm';
import { Classified } from './classified.js';

// Interfaz para seguridad de tipos
interface Reputation {
  id: number;
  sellerScore: number;
  buyerScore: number;
  sellerVotes: number;
  buyerVotes: number;
  user: User;
}

@Entity()
export class User extends BaseEntity {
  @PrimaryGeneratedColumn()
  id!: number;

  @Column({ unique: true })
  email!: string;

  @Column()
  name!: string;

  @Column()
  password!: string;

  @OneToMany(() => Classified, (classified) => classified.owner)
  classifieds!: Classified[];

  @Column({ type: 'varchar', nullable: true })
  verificationToken: string | null = null;

  @Column({ default: false })
  isVerified!: boolean;

  @Column({ type: 'varchar', nullable: true })
  resetToken: string | null = null;

  @Column({ type: 'varchar', nullable: true })
  pendingEmail: string | null = null;

  @Column({ type: 'varchar', nullable: true })
  changeEmailToken: string | null = null;

  @Column({
    type: process.env.NODE_ENV === 'test' ? 'datetime' : 'timestamp',
    nullable: true,
  })
  changeEmailTokenExpires: Date | null = null;

  @Column({
    type: process.env.NODE_ENV === 'test' ? 'datetime' : 'timestamp',
    nullable: true,
  })
  resetTokenExpires: Date | null = null;

  @OneToOne(
    'Reputation', // Nombre de la entidad
    (reputation: Reputation) => reputation.user,
    {
      cascade: true,
      eager: true,
    }
  )
  @JoinColumn()
  reputation!: Reputation;

  verifyEmail() {
    this.isVerified = true;
    this.verificationToken = null;
  }

  isEmailVerified(): boolean {
    return this.isVerified;
  }
}

Ejemplo para la entidad Reputation:
// src/models/reputation.ts
import { Entity, PrimaryGeneratedColumn, Column, OneToOne } from 'typeorm';

// Interfaz para seguridad de tipos
interface User {
  id: number;
  email: string;
  name: string;
  password: string;
  classifieds: any[];
  reputation: Reputation;
}

@Entity()
export class Reputation {
  @PrimaryGeneratedColumn()
  id!: number;

  @Column()
  sellerScore!: number;

  @Column()
  buyerScore!: number;

  @Column()
  sellerVotes!: number;

  @Column()
  buyerVotes!: number;

  @OneToOne(
    'User', // Nombre de la entidad
    (user: User) => user.reputation
  )
  user!: User;
}

Puntos clave:

Los decoradores @OneToOne('Reputation', ...) y @OneToOne('User', ...) usan nombres de entidades como cadenas, evitando importaciones directas.
Las interfaces (Reputation en user.ts y User en reputation.ts) proporcionan seguridad de tipos sin requerir importaciones.
El nombre de la entidad debe coincidir exactamente con el nombre de la clase o el valor pasado a @Entity() (si se personaliza).

2. Centralizar el Registro de Entidades
Crea un módulo para centralizar todas las entidades que se usarán en el DataSource de TypeORM. Esto asegura que TypeORM pueda resolver los nombres de las entidades en tiempo de ejecución.
// src/models/entities.ts
/**
 * Módulo que centraliza las entidades del modelo de datos de la aplicación.
 * @module entities
 */
import { User } from './user.js';
import { Reputation } from './reputation.js';
import { Classified } from './classified.js';

/**
 * Lista de entidades para la configuración del ORM.
 */
export const entityList = [User, Reputation, Classified];

3. Configurar el DataSource de TypeORM
Registra las entidades en el DataSource de TypeORM para que los nombres de las entidades se resuelvan correctamente.
// src/data-source.ts
import { DataSource } from 'typeorm';
import { entityList } from './models/entities.js';

export const AppDataSource = new DataSource({
  type: 'postgres', // Ajusta según tu base de datos
  host: process.env.DB_HOST,
  port: Number(process.env.DB_PORT),
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  entities: entityList,
  synchronize: process.env.NODE_ENV !== 'production',
});

Uso:Inicializa el DataSource en tu aplicación:
// src/index.ts
import { AppDataSource } from './data-source.js';

async function startApp() {
  await AppDataSource.initialize();
  console.log('Base de datos conectada');
  // Inicia tu aplicación
}

startApp().catch(console.error);

4. Opcional: Carga Dinámica de Entidades
Para proyectos con muchas entidades, mantener una lista manual en entities.ts puede ser tedioso. Puedes cargar las entidades dinámicamente usando import(), aunque esto requiere manejar importaciones asíncronas.
// src/data-source.ts
import { DataSource } from 'typeorm';

const entityPaths = [
  './models/user.js',
  './models/reputation.js',
  './models/classified.js',
];

async function loadEntities() {
  const entities = [];
  for (const path of entityPaths) {
    const module = await import(path);
    entities.push(module[Object.keys(module)[0]]);
  }
  return entities;
}

export const initializeDataSource = async () => {
  const entities = await loadEntities();
  return new DataSource({
    type: 'postgres',
    host: process.env.DB_HOST,
    port: Number(process.env.DB_PORT),
    username: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME,
    entities,
    synchronize: process.env.NODE_ENV !== 'production',
  });
};

Uso:
// src/index.ts
import { initializeDataSource } from './data-source.js';

async function startApp() {
  const dataSource = await initializeDataSource();
  await dataSource.initialize();
  console.log('Base de datos conectada');
  // Inicia tu aplicación
}

startApp().catch(console.error);

Nota: La carga dinámica es opcional y es más adecuada para proyectos con muchas entidades o cambios frecuentes.
Mejores Prácticas para Futuros Problemas

Usar Nombres de Entidades en Relaciones:

Utiliza cadenas (por ejemplo, 'Reputation', 'User') en los decoradores de relaciones para evitar importaciones circulares.
Asegúrate de que el nombre de la entidad coincida con el nombre de la clase o el valor de @Entity().


Mantener Seguridad de Tipos con Interfaces:

Define interfaces para las entidades relacionadas en cada archivo para garantizar la validación de tipos en TypeScript.
Mantén las interfaces mínimas, incluyendo solo las propiedades necesarias para la relación.


Centralizar el Registro de Entidades:

Usa un módulo único (por ejemplo, entities.ts) para listar todas las entidades que se registrarán en el DataSource.
Alternativamente, usa importaciones dinámicas para mayor escalabilidad.


Evitar Dependencias Globales:

No uses globalThis ni require, ya que introducen tipado débil y problemas de compatibilidad con ESM.


Documentar Entidades:

Usa comentarios JSDoc para documentar entidades, relaciones y métodos, mejorando la mantenibilidad.


Probar Relaciones:

Verifica que las relaciones bidireccionales (por ejemplo, @OneToOne) estén configuradas correctamente mediante pruebas de operaciones en la base de datos.
Usa eager: true o relaciones lazy con cuidado para evitar problemas de rendimiento.


Verificar Compatibilidad con ESM:

Asegúrate de que tu proyecto use ESM de manera consistente (por ejemplo, "type": "module" en package.json, extensiones .js en importaciones).


Consultar la Documentación de TypeORM:

Revisa la documentación de TypeORM para obtener orientación sobre relaciones y compatibilidad con ESM.



Errores Comunes

Nombres de Entidades Incorrectos: Asegúrate de que la cadena en @OneToOne('EntityName') coincida exactamente con el nombre de la clase o el valor de @Entity().
Entidades Faltantes en el DataSource: Todas las entidades deben estar registradas en el arreglo entities del DataSource, o TypeORM no podrá resolver las relaciones.
Pérdida de Tipos en TypeScript: Sin interfaces, el uso de nombres de entidades resulta en tipos any. Define siempre interfaces para mantener la seguridad de tipos.
Importaciones Asíncronas: Evita usar import() directamente en decoradores, ya que TypeORM espera referencias síncronas.

Conclusión
Al usar nombres de entidades en los decoradores de relaciones de TypeORM y interfaces para la seguridad de tipos, puedes resolver referencias circulares en proyectos ESM sin depender de globalThis ni require. Esta solución es limpia, segura en cuanto a tipos y compatible con proyectos TypeScript modernos. Para proyectos más grandes, considera la carga dinámica de entidades con import() en la configuración del DataSource.
