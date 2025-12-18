// gotham_game.c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#ifdef _WIN32
  #include <windows.h>
  void sleep_ms(int ms){ Sleep(ms); }
  void clear_screen(){ system("cls"); }
#else
  #include <unistd.h>
  void sleep_ms(int ms){ usleep(ms * 1000); }
  void clear_screen(){ printf("\033[2J\033[H"); }
#endif

// ====== STRUCTS

struct Heroi {
    char nome[32];
    int vida;
    int ataque;
    int defesa;
    int xp;
    int itens; // poções
    int nivel;
};

struct Vilao {
    char nome[32];
    int vida;
    int ataque;
};

// ====== PROTÓTIPOS ======
void animacao_pausa(const char *msg);
void mostrarStatus(struct Heroi h, struct Vilao v);
void lutar(struct Heroi *h, struct Vilao *v);
void espionagem(struct Heroi *h);
int charadasNivel1(struct Heroi *h);
int charadasNivel2(struct Heroi *h, struct Vilao *v);
int tentarPergunta(const char *pergunta, const char *opcoes[], int correta, int tentativas);
int venceu(int hp);
void nivelUp(struct Heroi *h);

// ====== IMPLEMENTAÇÕES ======

int venceu(int hp){
    return hp <= 0 ? 1 : 0;
}

void animacao_pausa(const char *msg){
    int i;
    printf("%s", msg);
    fflush(stdout);
    for(i=0;i<3;i++){
        sleep_ms(400);
        printf(".");
        fflush(stdout);
    }
    printf("\n");
}

void mostrarStatus(struct Heroi h, struct Vilao v){
    printf("\n--- STATUS ---\n");
    printf("%s | Vida: %d | Nivel: %d | XP: %d | Itens: %d\n", h.nome, h.vida, h.nivel, h.xp, h.itens);
    printf("%s | Vida: %d\n", v.nome, v.vida);
    printf("---------------\n");
}

// função de luta: inclui combos, itens, defender
void lutar(struct Heroi *h, struct Vilao *v){
    int opcao;
    int dano;
    int defendendo;

    while(h->vida > 0 && v->vida > 0){
        mostrarStatus(*h, *v);
        printf("1 - Soco (dano base 10)\n");
        printf("2 - Combo (dano base 30)\n");
        printf("3 - Batarangue (dano base 15)\n");
        printf("4 - Usar item (cura +30)\n");
        printf("5 - Defender (metade do dano inimigo neste turno)\n");
        printf("Escolha: ");
        if(scanf("%d", &opcao)!=1){ scanf("%s"); opcao = 0; }
        dano = 0;
        defendendo = 0;

        switch(opcao){
            case 1:
                printf("\nVoce deu um soco!\a\n");
                dano = 10 + (h->xp / 10);
                break;
            case 2:
                printf("\nVoce usou um combo devastador!\a\a\n");
                dano = 30 + (h->xp / 5);
                break;
            case 3:
                printf("\nVoce arremessou o Batarangue!\a\n");
                dano = 15 + (h->xp / 12);
                break;
            case 4:
                if(h->itens > 0){
                    h->vida += 30;
                    h->itens--;
                    printf("\nUsou um item! Vida +30\n");
                } else {
                    printf("\nSem itens!\n");
                }
                break;
            case 5:
                printf("\nVoce se defende!\n");
                defendendo = 1;
                break;
            default:
                printf("\nOpcao invalida! Voce perdeu a vez!\n");
                break;
        }

        if(dano > 0){
            v->vida -= dano;
            printf("Voce causou %d de dano a %s\n", dano, v->nome);
            if(venceu(v->vida)){
                printf("\n%s derrotado!\n", v->nome);
                h->xp += 25;
                animacao_pausa("Recolhendo experiencia");
                printf("\a"); // beep vitória
                return;
            }
        }

        // inimigo ataca se vivo
        if(v->vida > 0){
            int dano_inimigo = v->ataque;
            if(defendendo) dano_inimigo = (dano_inimigo + 1) / 2;
            // reduzir pelo atributo defesa do heroi (simples)
            int dano_final = dano_inimigo - (h->defesa / 4);
            if(dano_final < 1) dano_final = 1;
            h->vida -= dano_final;
            printf("%s atacou e causou %d de dano.\n", v->nome, dano_final);
            if(venceu(h->vida)){
                printf("\nBatman foi derrotado...\n");
                printf("\a\a"); // beep derrota
                return;
            }
        }
    }
}

// Espionagem: invadir ou espiar. Se invadir, enfrenta capangas e charada final (nvl2).
void espionagem(struct Heroi *h){
    int escolha;
    struct Vilao cap1 = {"Capanga Museu 1", 50, 18};
    struct Vilao cap2 = {"Capanga Museu 2", 60, 22};
    struct Vilao charada = {"O Charada (Nivel2)", 140, 30};

    printf("\n--- FASE DE ESPIONAGEM: MUSEU DE ASKHABAM ---\n");
    printf("Voce avista o Charada no museu.\n");
    printf("1 - Invadir (lutar contra capangas e depois charada)\n");
    printf("2 - Espionar (ganhar informacoes/itens e enfraquecer o charada)\n");
    printf("Escolha: ");
    if(scanf("%d", &escolha)!=1){ scanf("%*s"); escolha = 2; }

    if(escolha == 2){
        animacao_pausa("Espionando silenciosamente");
        printf("Voce obteve informacoes valiosas: +1 item e -20%% ataque do Charada final!\n");
        h->itens += 1;
        charada.ataque = (charada.ataque * 80)/100;
        h->xp += 15;
        return;
    }

    // invadir
    animacao_pausa("Invadindo o museu... cuidado");
    printf("Dois capangas surgem!\n");
    lutar(h, &cap1);
    if(h->vida <= 0) return;
    lutar(h, &cap2);
    if(h->vida <= 0) return;

    printf("\nVoce chegou no Charada! Ele propoe enigmas. Responda corretamente para vencê-lo.\n");
    int res = charadasNivel2(h, &charada);
    if(res){
        printf("\nVoce venceu o Charada! Gotham segura a respiração.\n");
        h->xp += 60;
        h->itens += 2;
    } else {
        printf("\nVoce falhou nas charadas. O Charada ataca ferozmente!\n");
        int golpe = charada.ataque * 2;
        h->vida -= golpe;
        if(h->vida <= 0) {
            printf("Batman foi derrotado pelo Charada.\n");
        } else {
            printf("Voce escapou ferido. Vida atual: %d\n", h->vida);
        }
    }
}

// pergunta com N tentativas: retorna 1 se acertou, 0 se esgotou
int tentarPergunta(const char *pergunta, const char *opcoes[], int correta, int tentativas){
    int r, i;
    for(i=0;i<tentativas;i++){
        printf("%s\n", pergunta);
        printf("1-%s  2-%s  3-%s  4-%s\n", opcoes[0], opcoes[1], opcoes[2], opcoes[3]);
        printf("Resposta (%d tentativas restantes): ", tentativas - i);
        if(scanf("%d", &r)!=1){ scanf("%*s"); r = 0; }
        if(r == correta){
            printf("Certo!\n");
            return 1;
        } else {
            printf("Errado!\n");
        }
    }
    return 0;
}

// Charadas Nível 1 (boss puzzle) — cada pergunta 3 tentativas
int charadasNivel1(struct Heroi *h){
    const char *op1[] = {"Toalha","Esponja","Gelo","Sabao"};
    if(!tentarPergunta("1) Quanto mais seca, mais molhada fica?", op1, 1, 3)) return 0;

    const char *op2[] = {"Foguete","Idade","Fumaca","Balao"};
    if(!tentarPergunta("2) O que sobe e nunca desce?", op2, 2, 3)) return 0;

    const char *op3[] = {"Mapa","Sonho","Quadro","Deserto"};
    if(!tentarPergunta("3) Cidades sem casas, rios sem agua. O que e?", op3, 1, 3)) return 0;

    // recompensa
    printf("Charada Nivel1 superada! Recompensa: XP +40 e 1 item\n");
    h->xp += 40;
    h->itens += 1;
    return 1;
}

// Charadas Nivel2 (mais dificil) com 3 perguntas, 3 tentativas cada
int charadasNivel2(struct Heroi *h, struct Vilao *v){
    const char *p1[] = {"Piano","Cofre","Cadeado","Caixa"};
    if(!tentarPergunta("1) O que tem chaves mas nao abre portas?", p1, 1, 3)) return 0;

    const char *p2[] = {"Nunca","Errado","Correto","Palavra"};
    if(!tentarPergunta("2) Qual palavra fica errada quando escrita corretamente?", p2, 2, 3)) return 0;

    const char *p3[] = {"Buraco","Fenda","Nuvem","Sombra"};
    if(!tentarPergunta("3) Quanto mais voce tira, maior ela fica. O que e?", p3, 1, 3)) return 0;

    // sucesso: enfraquece o vilao (se v passado), da xp e itens
    printf("Charada Nivel2 vencida! O charada fica atordoado e leva dano mental!\n");
    if(v) v->vida -= 40;
    h->xp += 80;
    h->itens += 2;
    return 1;
}

void nivelUp(struct Heroi *h){
    // cada 100 xp => +1 nivel (simplificado), ataque aumenta 20% por fase (chamado explicitamente)
    // apenas mostra mensagem se xp alta
    if(h->xp >= 100){
        h->nivel += 1;
        h->xp -= 100;
        // aumento percentual já gerenciado no fluxo principal
        printf("\n-- Nivel up! Agora nivel %d --\n", h->nivel);
        printf("\a");
    }
}

// ====== MAIN ======
int main(){
    struct Heroi batman;
    strcpy(batman.nome, "Batman");
    batman.vida = 150;
    batman.ataque = 30;
    batman.defesa = 18;
    batman.xp = 0;
    batman.itens = 2;
    batman.nivel = 1;

    // vilões das fases
    struct Vilao v1 = {"Ladrao da Loja", 45, 12};
    struct Vilao v2 = {"Capanga do Pinguim", 70, 16};
    struct Vilao v3 = {"Capanga do Coringa", 95, 20};
    struct Vilao v4 = {"Chefe das Docas", 120, 24};
    struct Vilao chefefase1 = {"Chefe Rua Final", 110, 26};
    struct Vilao chefefase2 = {"Coronel Pinguim", 150, 30};

    int opcao_menu;
    int fase = 1;

    // animação inicial
    clear_screen();
    animacao_pausa("Iniciando Gotham");
    printf("Bem vindo a Gotham, %s!\n", batman.nome);

    while(1){
        printf("\n====== MENU PRINCIPAL ======\n");
        printf("1 - Iniciar Fase %d\n", fase);
        printf("2 - Missao de Espionagem (Museu de Askhabam)\n");
        printf("3 - Ver status\n");
        printf("4 - Ir para Charadas Nivel 1 (boss de fase)\n");
        printf("5 - Sair\n");
        printf("Escolha: ");
        if(scanf("%d", &opcao_menu)!=1){ scanf("%*s"); opcao_menu = 5; }

        if(opcao_menu == 5){
            printf("Saindo... Ate a proxima.\n");
            break;
        }

        if(opcao_menu == 3){
            printf("\n%s | Vida: %d | Nivel: %d | XP: %d | Itens: %d | Ataque: %d | Defesa: %d\n",
                   batman.nome, batman.vida, batman.nivel, batman.xp, batman.itens, batman.ataque, batman.defesa);
            continue;
        }

        if(opcao_menu == 2){
            espionagem(&batman);
            if(batman.vida <= 0) { printf("Game over. Batman caiu.\n"); break; }
            nivelUp(&batman);
            continue;
        }

        if(opcao_menu == 4){
            if(!charadasNivel1(&batman)){
                printf("Falhou nas charadas. Gotham caiu.\n");
                break;
            } else {
                printf("Boss de fase derrotado por inteligencia!\n");
                nivelUp(&batman);
                // recompensa + subir ataque 20%
                int inc = (batman.ataque * 20) / 100;
                batman.ataque += inc;
                printf("Ataque aumentado em 20%% (%d)\n", batman.ataque);
            }
            continue;
        }

        if(opcao_menu == 1){
            // fases simples sequenciais com chefe final em cada conjunto
            if(fase == 1){
                printf("\n-- Fase 1: Ruas --\n");
                lutar(&batman, &v1);
                if(batman.vida <= 0) { printf("Volte mais forte.\n"); break; }
                printf("Agora enfrente o chefe da fase!\n");
                lutar(&batman, &chefefase1);
                if(batman.vida <= 0) { printf("Voce foi derrotado no chefe.\n"); break; }
                printf("Fase 1 COMPLETA!\n");
            } else if(fase == 2){
                printf("\n-- Fase 2: Docas --\n");
                lutar(&batman, &v2);
                if(batman.vida <= 0) { printf("Volte mais forte.\n"); break; }
                lutar(&batman, &v4);
                if(batman.vida <= 0) { printf("Voce foi derrotado.\n"); break; }
                printf("Chefe da fase aproximando...\n");
                lutar(&batman, &chefefase2);
                if(batman.vida <= 0) { printf("Voce foi derrotado no chefe.\n"); break; }
                printf("Fase 2 COMPLETA!\n");
            } else if(fase == 3){
                printf("\n-- Fase 3: Central Park --\n");
                lutar(&batman, &v3);
                if(batman.vida <= 0) { printf("Volte mais forte.\n"); break; }
                printf("Todos os capangas derrotados.\n");
                // agora proposta: ir para charada nivel2 final integrando espionagem
                printf("Proxima: Espionagem para enfrentar o Charada final (use opcao 2 do menu)\n");
            } else {
                printf("Todas as fases básicas completas. Use espionagem para atacar o Charada final.\n");
            }

            // ao completar cada fase (1 ou 2), subir nivel e +20% ataque
            if(fase <= 2){
                int incremento = (batman.ataque * 20) / 100;
                batman.ataque += incremento;
                batman.xp += 40;
                batman.nivel += 1;
                printf("Batman evoluiu! Ataque aumentado em 20%% (agora %d). XP +40\n", batman.ataque);
                printf("\a");
            }
            fase++;
            continue;
        }

        printf("Opcao invalida.\n");
    }

    printf("\nJogo finalizado. Obrigado!\n");
    return 0;
}
