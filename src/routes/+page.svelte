<script lang="ts">
    import {tweened} from "svelte/motion";
    import {blur} from "svelte/transition";
    import {onMount} from "svelte";
    import english from '../api/words/languages/english.json'

    type Game = 'in progress' | 'waiting for input' | 'game over'
    type Word = string


    let game: Game = 'waiting for input'
    let typedLetter = ''
    let wordIndex = 0
    let letterIndex = 0
    let correctLetters = 0
    let typedLetters = 0
    let wordCount = 0
    let seconds = 0
    let totalSeconds = seconds
    let toggleReset = false

    let totalWords: Word[] = []
    totalWords = english.words
    let words: Word[] = getRandomWords(totalWords, wordCount)

    let wpm = tweened(0, {delay: 300, duration: 500})
    let accuracy = tweened(0, {delay: 300, duration: 500})
    let lps = tweened(0, {delay: 300, duration: 500})

    let wordsElement: HTMLDivElement
    let caretElement: HTMLDivElement
    let letterElement: HTMLSpanElement
    let inputElement: HTMLInputElement

    // Functions
    function getRandomWords(words: Word[], wordCount: number) {
        return words.sort(() => Math.random() - 0.5).slice(0, wordCount)
    }
    function resetGame() {
        toggleReset = !toggleReset
        setGameState('waiting for input')
        words = getRandomWords(totalWords, wordCount)
        seconds = totalSeconds
        typedLetter = ''
        wordIndex = 0
        letterIndex = 0
        correctLetters = 0
        $wpm = 0
        $accuracy = 0
    }
    function getWPM() {
        const time = totalSeconds - seconds
        $lps = (correctLetters / time)
        return Math.floor(($lps * 60) / 4.5)
    }
    function getAccuracy() {
        return Math.floor((correctLetters / typedLetters) * 100)
    }
    function getResults() {
        $wpm = getWPM()
        $accuracy = getAccuracy()
    }
    function startGame() {
        setGameState('in progress')
        setGameTimer()
    }
    function setGameTimer() {
        function gameTimer() {
            if(seconds > 0) {
                seconds -= 1
            }
            if(game === 'waiting for input' || seconds === 0) {
                clearInterval(interval)
            }
            if(seconds === 0) {
                setGameState('game over')
                getResults()
            }
        }
        const interval = setInterval(gameTimer, 1000)
    }
    function setGameState(state: Game) {
        game = state
    }
    function updateGameState() {
        if(wordIndex === words.length - 1 && letterIndex === words[wordIndex].length - 1) {
            setGameState('game over')
            getResults()
        }
        setLetter()
        checkLetter()
        nextLetter()
        updateLine()
        resetLetter()
        moveCaret()
    }
    function setLetter() {
        const isWordCompleted = letterIndex > words[wordIndex].length - 1
        if(!isWordCompleted) {
            letterElement = wordsElement.children[wordIndex].children[letterIndex] as HTMLSpanElement
        }
    }
    function checkLetter() {
        const currLetter = words[wordIndex][letterIndex]
        typedLetters += 1
        if(typedLetter === currLetter) {
            letterElement.dataset.letter = 'correct'
            increaseScore()
        } else {
            letterElement.dataset.letter = 'incorrect'
        }
    }
    function increaseScore() {
        correctLetters += 1
    }
    function nextLetter() {
        letterIndex += 1
    }
    function nextWord() {
        const isNotFirstLetter = letterIndex !== 0
        const isOneLetterWord = words[wordIndex].length === 1
        if(isNotFirstLetter || isOneLetterWord) {
            wordIndex += 1
            letterIndex = 0
            moveCaret()
        }
    }
    function updateLine() {
        const wordElement = wordsElement.children[wordIndex]
        const wordsY = wordsElement.getBoundingClientRect().y
        const wordY = wordElement.getBoundingClientRect().y

        if(wordY > wordsY) {
            wordElement.scrollIntoView({ block: 'center', behavior: 'smooth' })
        }
    }
    function resetLetter() {
        typedLetter = ''
    }
    function moveCaret() {
        const { offsetTop, offsetLeft, offsetWidth } = letterElement
        caretElement.style.top = `${offsetTop}px`
        caretElement.style.left = `${offsetLeft + offsetWidth}px`
    }
    function focusInput() {
        inputElement.focus()
    }
    function handleKeyDown (event: KeyboardEvent) {
        if(event.code === 'Space') {
            event.preventDefault()
            if(game === 'in progress') {
                nextWord()
            }
        }
        if(game === 'waiting for input') {
            startGame()
        }
    }
    onMount(() => {
        focusInput()
    })
</script>

{#if game === 'game over'}
    <div class="results" in:blur|local>
        <div class="result">
            <p class="title">WPM</p>
            <p class="score">{Math.trunc($wpm)}</p>
        </div>
        <div class="result">
            <p class="title">LPS</p>
            <p class="score">{Math.trunc($lps)}</p>
        </div>
        <div class="result">
            <p class="title">accuracy</p>
            <p class="score">{Math.trunc($accuracy)}%</p>
        </div>
        <button class="play" on:click={resetGame}>Play Again</button>
    </div>
{:else}
    <div class="game" data-game={game}>
        {#if game==='waiting for input'}
            <div class="menubar">
                <div class="form__group">
                    <label class="form__label">Time</label>
                    <input
                            class="form__field"
                            type="number"
                            bind:value={totalSeconds}
                            on:input={resetGame}
                    />
                </div>
                <div class="form__group">
                    <label class="form__label">Word Count</label>
                    <input
                            class="form__field"
                            type="number"
                            bind:value={wordCount}
                            on:input={resetGame}
                    />
                </div>
            </div>
        {:else}
            <div class="timer">
                {seconds}
            </div>
        {/if}
        <input
                bind:this={inputElement}
                bind:value={typedLetter}
                on:input={updateGameState}
                on:keydown={handleKeyDown}
                type="text"
                class="input"
        />
        {#key toggleReset}
            <div on:click={focusInput}>
                <div in:blur|local class="words" bind:this={wordsElement}>
                    {#each words as word}
                        <span class="word">
                            {#each word as letter}
                                <span class="letter">
                                    {letter}
                                </span>
                            {/each}
                        </span>
                    {/each}
                    <div class="caret" bind:this={caretElement}></div>
                </div>
            </div>
        {/key}

        <div class="reset">
            <button on:click={resetGame}>Reset</button>
        </div>
    </div>
{/if}

<!--Style-->
<style lang="scss">
      input[type=number]::-webkit-inner-spin-button,
      input[type=number]::-webkit-outer-spin-button {
        -webkit-appearance: none;
        margin: 0;
      }
    .game {
      display: flex;
      flex-direction: column;
      align-items: center;
      gap: 2rem;
      justify-content: space-evenly;
      position: relative;
      .menubar {
        display: flex;
        flex-direction: row;
        justify-content: space-evenly;
        $primary: #11998e;
        $secondary: #38ef7d;
        $white: #fff;
        $gray: #9b9b9b;

        .form__group {
          position: relative;
          padding: 15px 0 0;
          margin-top: 10px;
          width: 50%;

        .form__field {
          font-family: inherit;
          width: 100%;
          border: 0;
          //border-bottom: 2px solid $gray;
          outline: 0;
          font-size: 1.3rem;
          color: $white;
          padding: 7px 0;
          background: transparent;
          transition: border-color 0.2s;

          &::placeholder {
            color: transparent;
          }

          &:placeholder-shown ~ .form__label {
            font-size: 1.3rem;
            cursor: text;
            top: 20px;
          }
        }

        .form__label {
          position: absolute;
          top: 0;
          display: block;
          transition: 0.2s;
          font-size: 1rem;
          color: $gray;
        }

        .form__field:focus {
          ~ .form__label {
            position: absolute;
            top: 0;
            display: block;
            transition: 0.2s;
            font-size: 1rem;
            color: $primary;
            font-weight: 700;
          }

          padding-bottom: 6px;
          font-weight: 700;
          border-width: 3px;
          border-image: linear-gradient(to right, $primary, $secondary);
          border-image-slice: 1;
        }

        /* reset input */
        .form__field {
          &:required, &:invalid {
            box-shadow: none;
          }
        }
      }
      }
          &[data-game='in progress'] .caret {
            animation: none;
          }
          &[data-game='in progress'] .timer {
            opacity: 1;
          }
          .input {
            position: absolute;
            opacity: 0;
          }
          .timer {
            position: absolute;
            top: -48px;
            font-size: 1.5rem;
            color: var(--primary);
            opacity: 0;
            transition: all 0.3s ease;
          }

      .reset {
        width: 100%;
        display: grid;
        justify-content: center;
        margin-top: 2rem;
      }
    }
    .results {
      display: flex;
      flex-direction: column;
      justify-content: space-between;
      align-items: center;
      .result {
        display: flex;
        flex-direction: column-reverse;
        align-items: center;
        justify-content: center;
        gap: 1em;
      }
      .title {
        font-size: 2rem;
        color: var(--fg-200);
      }

      .score {
        font-size: 4rem;
        color: var(--primary);
      }

      .play {
        margin-top: 1rem;
      }
    }
    .words {
        --line-height: 1em;
        --lines: 3;

        width: 100%;
        max-height: calc(var(--line-height) * var(--lines) * 1.42);
        display: flex;
        flex-wrap: wrap;
        justify-content: center;
        gap: 0.6em;
        position: relative;
        font-size: 1.5rem;
        line-height: var(--line-height);
        overflow: hidden;
        user-select: none;

        .caret {
          position: absolute;
          height: 1.8rem;
          top: 0;
          border-right: 1px solid var(--primary);
          animation: caret 1s infinite;
          transition: all 0.2s ease;

          @keyframes caret {
            0%,
            to {
              opacity: 0;
            }
            50% {
              opacity: 1;
            }
          }
        }
        .letter {
          opacity: 0.4;
          transition: all 0.3s ease;

          &:global([data-letter='correct']) {
            opacity: 0.8;
          }

          &:global([data-letter='incorrect']) {
            color: var(--primary);
            opacity: 1;
          }
        }
    }
</style>
