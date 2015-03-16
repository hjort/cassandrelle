# O que é isso? #

O componente **Demoiselle Cassandra** (aka **Cassandrelle**) fornece uma interface simplificada e altamente orientada a objetos para lidar com dados armazenados no banco **Apache Cassandra**.

# Exemplo de uso #

1. Crie a classe correspondente à entidade e anote com `@CassandraEntity` e `@KeyProperty`:

```
@CassandraEntity(keyspace = "Twitter", columnFamily = "Users")
public class User {

        @KeyProperty
        private String login;
        
        private String name;
        private String password;

        ...
}
```

2. Crie uma interface estendendo a interface `CassandraDAO` e adicione os métodos específicos da regra de negócio:

```
public interface IUserDAO extends CassandraDAO<User> {

        User findByLogin(String login);

        List<User> findByLogins(Iterable<String> logins);

}
```

3. Crie uma classe implementando a interface anterior derivando da superclasse `CassandraEntityDAO`:

```
public class UserDAO extends CassandraEntityDAO<User> implements IUserDAO {
        
        public User findByLogin(String login) {
                return get(login);
        }

        public List<User> findByLogins(Iterable<String> logins) {
                return get(logins);
        }

}
```

4. Crie uma classe implementando a interface `IFacade`:

```
public class TwitterFacade extends IFacade {

        @Injection
        private IUserDAO userDAO;

        public User createUser(String login, String name, String password) {
                
                User user = new User();
                user.setLogin(login);
                user.setName(name);
                user.setPassword(password);
                
                userDAO.save(user);
                
                return user;
        }

        public void removeUser(String login) {
                userDAO.delete(new User(login));
        }

        public User findUserByLogin(String login) {
                return userDAO.findByLogin(login);
        }

}
```