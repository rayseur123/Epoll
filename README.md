# Epoll

## Description

Ce guide rassemble mes notes sur epoll, avec pour objectif personnel de centraliser l'information et de pouvoir l'expliquer clairement.

---

## select() / pselect()

Ces fonctions surveillent un ensemble de file descriptors (fd) et indiquent lesquels sont disponibles en lecture, écriture ou en erreur.

Différences entre `select` et `pselect` :

- `select` utilise `timeval` (secondes, microsecondes) ; `pselect` utilise `timespec` (secondes, nanosecondes).
- `select` peut modifier le paramètre `timeout` pour indiquer le temps restant.
- `pselect` accepte un `sigmask` (sigset) définissant les signaux à ignorer pendant l'attente.

Déclarations :

```c
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);

int pselect(int nfds, fd_set *readfds, fd_set *writefds,
            fd_set *exceptfds, const struct timespec *timeout,
            const sigset_t *sigmask);
```

Les trois ensembles de fd sont surveillés simultanément. À la sortie, ils ne contiennent que les fd dont l'état a changé (ceux devenus disponibles). Si aucun fd n'est disponible, l'ensemble correspondant est mis à NULL.

- `readfds` : fd disponibles en lecture
- `writefds` : fd disponibles en écriture
- `exceptfds` : fd en exception (données urgentes / out-of-band / erreur)

`nfds` est le numéro du fd le plus élevé parmi les trois ensembles, plus 1. Les `fd_set` étant des tableaux de bits de taille fixe (1024), ce paramètre limite la recherche à l'intervalle `[0, nfds[` plutôt que `[0, 1024[`, réduisant le nombre d'itérations.

`timeout` définit le délai maximum d'attente si aucun fd n'est disponible.

Macros disponibles pour manipuler les `fd_set` :

```
FD_ZERO(fd_set *set)           – initialise l'ensemble
FD_SET(int fd, fd_set *set)    – ajoute fd à l'ensemble
FD_CLR(int fd, fd_set *set)    – retire fd de l'ensemble
FD_ISSET(int fd, fd_set *set)  – teste si fd est dans l'ensemble
```

---

## poll() / ppoll()

Même vocation que `select()`/`pselect()`, mêmes différences entre les deux variantes. La structure utilisée est différente : un tableau de `pollfd` représente les trois types de fd en un seul paramètre.

Déclarations :

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

int ppoll(struct pollfd *fds, nfds_t nfds,
          const struct timespec *timeout, const sigset_t *sigmask);
```

Ici, `nfds` est simplement la taille du tableau passé en paramètre (et non le fd maximum + 1 comme dans `select`).

---

## epoll

epoll est une API (ensemble de fonctions) conçue pour rester performante quel que soit le nombre de fd gérés, là où `poll` dégrade avec la montée en charge.

### epoll_create()

Crée un fd pointant vers un espace de surveillance. Le paramètre `nb` donne une indication de taille au noyau pour optimiser ses structures internes (non contraignant).

```c
int epoll_create(int nb);
```

### epoll_ctl()

Applique l'opération `op` sur le fd `fd` dans l'ensemble `epfd`.

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

Opérations disponibles :

```
EPOLL_CTL_ADD  – ajoute fd dans epfd
EPOLL_CTL_MOD  – modifie l'événement associé à fd
EPOLL_CTL_DEL  – retire fd de epfd
```

Le dernier paramètre est un masque de bits listant les événements à surveiller pour ce fd :

```
EPOLLIN      – fd disponible en lecture
EPOLLOUT     – fd disponible en écriture
EPOLLRDHUP   – le client a fermé la connexion (côté serveur)
EPOLLPRI     – données urgentes disponibles en lecture
EPOLLERR     – erreur sur le fd (toujours actif, même sans le spécifier)
EPOLLHUP     – le fd s'est déconnecté
EPOLLET      – active le mode edge-triggered (notification uniquement au changement d'état)
EPOLLONESHOT – désactive le fd dans epfd après un événement (nécessite un EPOLL_CTL_MOD pour le réarmer)
```

### epoll_wait() / epoll_pwait()

Attendent qu'un événement se produise sur un fd de l'ensemble `epfd` et remplissent le tableau `events` avec les événements survenus.

```c
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);

int epoll_pwait(int epfd, struct epoll_event *events,
                int maxevents, int timeout,
                const sigset_t *sigmask);
```
