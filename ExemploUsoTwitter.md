# Modelo de dados #

Para ilustrar a modelagem de dados orientada a colunas fornecida pelo **Apache Cassandra**, um grande exemplo são as _"aplicações sociais"_, tais como **Twitter** ou **Google Buzz**.

As famílias de colunas (column families) envolvidas estão ilustradas a seguir:

![http://demoiselle.sourceforge.net/component/demoiselle-cassandra/1.0.0-RC1/images/twitter-model1.png](http://demoiselle.sourceforge.net/component/demoiselle-cassandra/1.0.0-RC1/images/twitter-model1.png)

  * **Users**: usuários cadastrados na aplicação
  * **Following**: lista de pessoas seguidas para cada usuário
  * **Followers**: lista de seguidores para cada usuário

![http://demoiselle.sourceforge.net/component/demoiselle-cassandra/1.0.0-RC1/images/twitter-model2.png](http://demoiselle.sourceforge.net/component/demoiselle-cassandra/1.0.0-RC1/images/twitter-model2.png)

  * **Tweets** (ou **Statuses**): lista de mensagens (ou tweets)
  * **Timeline**: linha do tempo para cada usuário, ou seja, lista de mensagens de todos as pessoas que ele segue
  * **Userline**: linha de cada usuário, isto é, lista de mensagens incluídas pelo próprio usuário

# Persistência de entidade (Users) #

## Criação da entidade User ##

A classe correspondente à entidade `User` armazenará objetos Java que serão inteiramente persistidos na família de colunas `Users` pertencente ao keyspace `Twitter` do **Apache Cassandra**.

Para implementar essa classe, são usadas as anotações `@CassandraEntity` (para definir que a primeira trata-se de uma entidade) e `@KeyProperty` (para definir a propriedade referente à chave na família de colunas).

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

## Criação da interface IUserDAO ##

Neste caso, é preciso criar a interface `IUserDAO` estendendo a interface genérica `CassandraDAO` e na primeira adicionar os métodos específicos da regra de negócio:

```
public interface IUserDAO extends CassandraDAO<User> {

        User findByLogin(String login);

        List<User> findByLogins(Iterable<String> logins);

}
```

## Criação da classe UserDAO ##

A classe `UserDAO` trata-se de uma implementação da interface `IUserDAO`, além de derivar da superclasse `CassandraEntityDAO`, a qual fornece métodos para a manipulação dos dados:

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

## Criação da classe TwitterFacade ##

A classe `TwitterFacade` implementa a interface `IFacade`, que por sua vez torna possível a injeção de dependências de variáveis anotadas com `@Injection`.

Veja no código a seguir a implementação da classe `TwitterFacade` com seus respectivos métodos para criação, remoção e busca de usuários.

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

# Persistência de coluna dupla (Following, Followers) #

Neste outro caso, uma informação precisa ser persistida de forma diferente em duas famílias de colunas, `Following` e `Followers`. O que difere uma de outra é a indexação, ou seja, como o dado será disposto na família de colunas.

## Criação da entidade Followship ##

A classe `Followship` armazenará a ligação entre um usuário (o seguidor, ou follower) a um outro (o seguido, ou followed). Para isso, é preciso utilizar a anotação `@CassandraColumn`, indicando nesta as famílias de colunas primária (`Following`) e secundária (`Followers`). Essa anotação também define o keyspace a ser utilizado.

As propriedades devem ser anotadas com `@KeyProperty` e `@ColumnProperty`.

```
@CassandraColumn(keyspace = "Twitter", columnFamily = "Following",
                secondaryColumnFamily = "Followers")
public class Followship {

        @KeyProperty
        private String follower;

        @ColumnProperty
        private String followed;

        ...
}
```

## Criação da interface IFollowshipDAO ##

Tal como anteriormente, é preciso criar a interface `IFollowshipDAO` estendendo a interface genérica `CassandraDAO` e na primeira adicionar os métodos específicos da regra de negócio:

```
public interface IFollowshipDAO extends CassandraDAO<Followship> {

        List<String> findFollowingsLogins(String follower);

        List<String> findFollowersLogins(String followed);

}
```

## Criação da classe FollowshipDAO ##

A classe `FollowshipDAO` também trata-se de uma implementação da interface `CassandraDAO`, porém deriva da superclasse `CassandraColumnDAO`, a qual fornece métodos para a manipulação dos dados no formato de colunas:

```
public class FollowshipDAO extends CassandraColumnDAO<Followship>
                implements IFollowshipDAO {

        public List<String> findFollowingsLogins(String follower) {
                return getColumns(follower);
        }

        public List<String> findFollowersLogins(String followed) {
                return getColumnsBySecondary(followed);
        }

}
```

## Modificação da classe TwitterFacade ##

A classe `TwitterFacade` deve ser modificada para poder realizar operações como follow, unfollow e recuperação de lista de seguidos e seguidores de um usuário.

```
public class TwitterFacade extends IFacade {

        ...
    
        @Injection
        private IFollowshipDAO followshipDAO;
    
        ...
    
        public Followship followUser(String login, String followed) {

                Followship followship = new Followship();
                followship.setFollower(login);
                followship.setFollowed(followed);
                
                followshipDAO.save(followship);

                return followship;
        }

        public void unfollowUser(String login, String followed) {

                Followship followship = new Followship();
                followship.setFollower(login);
                followship.setFollowed(followed);
                
                followshipDAO.delete(followship);
        }

        public List<User> findUserFollowings(String login) {
                
                List<String> ids = followshipDAO.findFollowingsLogins(login);
                
                if (ids == null || ids.isEmpty())
                        return null;
                
                List<User> users = userDAO.findByLogins(ids);
                
                return users;
        }

        public List<User> findUserFollowers(String login) {
                
                List<String> ids = followshipDAO.findFollowersLogins(login);

                if (ids == null || ids.isEmpty())
                        return null;
                
                List<User> users = userDAO.findByLogins(ids);
                
                return users;
        }

}
```

# Persistência de entidade e colunas (Tweets, UserLine, TimeLine) #

## Criação da entidade Tweet ##

É preciso criar a entidade `Tweet` destinada a armazenar as mensagens emitidas pelos usuários. Neste caso, será empregada a anotação `@CassandraEntity` indicando a família de colunas `Tweets` e `@KeyProperty` para definir o campo a ser usado como chave.

```
@CassandraEntity(keyspace = "Twitter", columnFamily = "Tweets")
public class Tweet {

        @KeyProperty
        private Long id;
        
        private String user;
        private String text;

        ...
}
```

A fim de poder futuramente recuperar a lista de mensagens incluídas por um usuário, precisamos de uma estrutura adicional, a família de colunas `Userline`, a qual será indexada pelo respectivo usuário.

Para isso, crie a classe `UserLine` utilizando nela a anotação `@CassandraColumn` e em seus campos as anotações `@KeyProperty`, `@ColumnProperty` e `@ValueProperty`.

```
@CassandraColumn(keyspace = "Twitter", columnFamily = "Userline")
public class UserLine implements TweetLine {

        @KeyProperty
        private String user;
        
        @ColumnProperty
        private Long time;
        
        @ValueProperty
        private Long tweet;
        
        ...
}
```

Para armazenar a lista de mensagens incluídas pelos usuários que alguém segue, é preciso ainda de uma terceira estrutura, a família de colunas `Timeline`, a qual será indexada pelo usuário dito seguidor.

Crie a classe `TimeLine` fazendo uso da anotação `@CassandraColumn` e em suas propriedades as anotações `@KeyProperty`, `@ColumnProperty` e `@ValueProperty`.

```
@CassandraColumn(keyspace = "Twitter", columnFamily = "Timeline")
public class TimeLine implements TweetLine {

        @KeyProperty
        private String user;

        @ColumnProperty
        private Long time;
        
        @ValueProperty
        private Long tweet;

        ...
}
```

## Criação das interfaces ITweetDAO, IUserLineDAO e ITimeLineDAO ##

É preciso criar as interfaces `ITweetDAO`, `IUserLineDAO` e `ITimeLineDAO` estendendo a interface genérica `CassandraDAO` e nelas adicionar os métodos específicos da regra de negócio:

```
public interface ITweetDAO extends CassandraDAO<Tweet> {

        Tweet findById(Long id);

        List<Tweet> findByIds(Iterable<Long> ids);

}
```

```
public interface IUserLineDAO extends CassandraDAO<UserLine> {

        List<Long> findUserLine(String user);

}
```

```
public interface ITimeLineDAO extends CassandraDAO<TimeLine> {
        
        List<Long> findTimeLine(String user);

}
```

## Criação das classes TweetDAO, UserLineDAO e TimeLineDAO ##

A classe `FollowshipDAO` também trata-se de uma implementação da interface `CassandraDAO`, porém deriva da superclasse `CassandraColumnDAO`, a qual fornece métodos para a manipulação dos dados no formato de colunas:

```
public class TweetDAO extends CassandraEntityDAO<Tweet> implements ITweetDAO {

        public Tweet findById(Long id) {
                return get(id.toString());
        }

        public List<Tweet> findByIds(Iterable<Long> ids) {
                return get(Iterables.transform(ids, Functions.toStringFunction()));
        }

}
```

A classe `UserLineDAO` também trata-se de uma implementação da interface `CassandraDAO`, porém deriva da superclasse `CassandraColumnDAO`, a qual fornece métodos para a manipulação dos dados no formato de colunas:

```
public class UserLineDAO extends CassandraColumnDAO<UserLine>
                implements IUserLineDAO {

        public List<Long> findUserLine(String user) {
                
                List<String> values = getValues(user);
                
                if (values == null || values.isEmpty())
                        return null;
                
                List<Long> tweets = Lists.transform(values,
                                new Function<String, Long>() {
                        public Long apply(String from) {
                                return Long.parseLong(from);
                        }
                });
                
                return tweets;
        }

}
```

A classe `TimeLineDAO` também trata-se de uma implementação da interface `CassandraDAO`, porém deriva da superclasse `CassandraColumnDAO`, a qual fornece métodos para a manipulação dos dados no formato de colunas:

```
public class TimeLineDAO extends CassandraColumnDAO<TimeLine>
                implements ITimeLineDAO {

        public List<Long> findTimeLine(String user) {
                
                List<String> values = getValues(user);
                
                if (values == null || values.isEmpty())
                        return null;
                
                List<Long> tweets = Lists.transform(values,
                                new Function<String, Long>() {
                        public Long apply(String from) {
                                return Long.parseLong(from);
                        }
                });
                
                return tweets;
        }

}
```

## Modificação da classe TwitterFacade ##

A classe `TwitterFacade` deve ser modificada para poder realizar operações como postagem (inclusão de mensagem), remoção de mensagem, busca de mensagem e listagem de mensagens para determinado usuário.

```
public class TwitterFacade extends IFacade {

        ...
    
        @Injection
        private ITweetDAO tweetDAO;
        
        @Injection
        private IUserLineDAO userlineDAO;
        
        @Injection
        private ITimeLineDAO timelineDAO;
    
        ...

        public Tweet postTweet(String login, String text) {

                final long id = (long) (Math.random() * 1E10);

                Tweet tweet = new Tweet();
                tweet.setId(id);
                tweet.setUser(login);
                tweet.setText(text);
                
                tweetDAO.save(tweet);
                
                final long timestamp = System.currentTimeMillis();

                UserLine userline = new UserLine();
                userline.setUser(login);
                userline.setTime(timestamp);
                userline.setTweet(id);
                userlineDAO.save(userline);

                List<String> logins = followshipDAO.findFollowersLogins(login);
                if (logins != null) {
                        TimeLine timeline = null;
                        for (String follower : logins) {
                                timeline = new TimeLine();
                                timeline.setUser(follower);
                                timeline.setTime(timestamp);
                                timeline.setTweet(id);
                                timelineDAO.save(timeline);
                        }
                }

                return tweet;
        }

        public void removeTweet(Long id) {
                Tweet tweet = tweetDAO.findById(id);
                tweetDAO.delete(tweet);
        }

        public Tweet findTweet(Long id) {
                Tweet tweet = tweetDAO.findById(id);
                return tweet;
        }

        public List<Tweet> findUserLastTweets(String login, int count) {

                List<Long> ids1 = userlineDAO.findUserLine(login);
                List<Long> ids2 = timelineDAO.findTimeLine(login);
                
                List<Long> ids = new ArrayList<Long>();
                if (ids1 != null)
                        ids.addAll(ids1);
                if (ids2 != null)
                        ids.addAll(ids2);
                
                if (ids == null || ids.isEmpty())
                        return null;
                
                List<Tweet> tweets = tweetDAO.findByIds(ids);

                return tweets;
        }

        public List<Tweet> findUserLastTweets(String login) {
                return findUserLastTweets(login, TWEETS_DEFAULT_COUNT);
        }
        
}
```