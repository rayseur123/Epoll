# Epoll

## Description

Ce guide me permet simplement de rassembler la totalité de mes informations sur epoll. Il a donc un objectif personnel mais me permettra aussi d'expliquer de façon plus aisée le fonctionnement de epoll en détail.

## select() / pselect()

Avant de comprendre l'intérêt de epoll il faut se pencher sur ses prédécesseurs.
Ces fonctions ont pour objectif de surveiller et d'indiquer si un fd (file descriptor) est disponible à la lecture / écriture.

Quelques différences entre select() / pselect() :

- `select` utilise la structure `timeval` (secondes, microsecondes) pour représenter le temps.
- `pselect` utilise la structure `timespec` (secondes, nanosecondes) pour représenter le temps.
- `select` peut modifier le paramètre timeout pour indiquer le temps restant.
- `pselect` prend en paramètre `sigmask` (sigset) représentant l'ensemble des signaux à ignorer, sans ça select peut être bloquant.

Voici les déclarations de ces fonctions :

```c
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);

int pselect(int nfds, fd_set *readfds, fd_set *writefds,
            fd_set *exceptfds, const struct timespec *timeout,
            const sigset_t *sigmask);
```

Nous leur passons trois ensembles indépendants de descripteurs à surveiller simultanément. Ces ensembles seront alors modifiés par la fonction pour ne garder que ce qui change d'état, c'est-à-dire ceux devenant disponibles pour leur fonction.

Si aucun fd n'est disponible, alors l'ensemble concerné sera mis à NULL.

- `readfds` représente les fd de lecture
- `writefds` représente les fd d'écriture
- `exceptfds` représente les fd d'exception (urgent / out-of-band / erreur)

`nfds` est le numéro du plus grand descripteur de fichier des 3 ensembles, plus 1.
**Pourquoi ?** Les fd_set étant stockés dans un tableau de bits (bitfield) de taille fixe (1024). Sans ce paramètre, la totalité de ces bits devraient être surveillée. Alors que là, la recherche est `0 - ndfs` plutôt que `0 - 1024`. Ce qui réduit grandement le nombre d'itérations.

`timeout` représente le temps max que select prendra si aucun fd n'est disponible.

En résumé, select renvoie dans les trois fd set les fd disponibles à l'usage sur le moment. Si aucun n'est disponible, select/pselect attendra jusqu'au timeout si besoin.

Certaine macro sont disponible pour editer les fd_set.
```
- FD_ZERO(fd_set *set) - initialise l'ensemble
- FD_SET(int fd, fd_set *set) - Ajoute fd à l'ensemble
- FD_CLR(int fd, fd_set *set) - Elimine fd de l'ensemble
- FD_ISSET(int fd, fd_set *set) - test pour savoir si fd fait parti de l'ensemble
```

## poll() / ppoll()
Avant tout, voici la déclaration de ces deux fonctions :
```
int poll(struct pollfd *fds, nfds_t nfds, int délai);

int ppoll(struct pollfd *fds, nfds_t nfds, 
        const struct timespec *délai, const sigset_t *sigmask);
```
Ces fonctions ont la même vocation que select()/pselect(), leurs différences sont donc similaires. A la différence qu'ils ne prennent qu'un tableau de structure `pollfd` représentant les trois types de fd.
structure de pollfd :
```
struct pollfd {
    int   fd;         /* Descripteur de fichier */
    short events;     /* Événements attendus    */
    short revents;    /* Événements détectés    */
};
```

Aussi, le parametre `nfds` ne signifie pas la meme chose. nfds represente ici simplement la taille du tableau passé en parametre.

## epoll()
epoll() est une variante de poll qui a pour vocation de rester optimiser malgré le grand nombre de fd géré.
