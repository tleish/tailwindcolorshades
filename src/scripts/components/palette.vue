<script>
import convert from 'color-convert'

const CMY_HUES = [180, 300, 60];
const RGB_HUES = [360, 240, 120, 0];

export default {
  name: "palette",

  props: {
    name: String,
    color: String,
    version: {
      type: Number,
      default: 1
    }
  },

  created() {
    this.colorName = this.name;
  },

  filters: {
    unquoted(text) {
      return text.replace(/'(.*?)'/g, "$1");
    }
  },

  data() {
    return {
      newName: "",

      colorName: "",

      isRenaming: false,

      tintsPerVersion: {
        1: {
          100: 0.8,
          200: 0.6,
          300: 0.4,
          400: 0.2
        },
        0: {
          lightest: 0.8,
          lighter: 0.5,
          light: 0.2
        }
      },

      shadesPerVersion: {
        1: {
          600: 0.2,
          700: 0.4,
          800: 0.6,
          900: 0.8
        },
        0: {
          dark: 0.2,
          darker: 0.5,
          darkest: 0.8
        }
      },

      colors: []
    };
  },

  mounted() {
    this.$nextTick(() => {
      this.generate();
    });
  },

  methods: {
    remove() {
      this.$emit("remove");
    },

    hueShift(hues, hue, intensity) {
      const closestHue = hues.sort((a, b) => (Math.abs(a - hue) - Math.abs(b - hue)))[0],
            hueShift = closestHue - hue;
      return Math.round(intensity * hueShift * 0.25)
    },

    tint(hex, intensity) {
      const [h,s,v] = convert.hex.hsv(hex),
        hNew = h + this.hueShift(CMY_HUES, h, intensity),
        sNew = s - Math.round(s * intensity),
        vNew = v + Math.round((100 - v) * intensity)
      ;
      return `#${convert.hsv.hex(hNew, sNew, vNew)}`;
    },

    shade(hex, intensity) {
      const [h,s,v] = convert.hex.hsv(hex),
        hNew = h + this.hueShift(RGB_HUES, h, intensity),
        sNew = s + Math.round((100 - s) * intensity),
        vNew = v - Math.round(v * intensity)
      ;
      return `#${convert.hsv.hex(hNew, sNew, vNew)}`;
    },

    getTextColor(color) {
      const [ r, g, b ] = convert.hex.rgb(color),
        luma = 0.2126 * r + 0.7152 * g + 0.0722 * b;

      return luma < 120 ? "#FFFFFF" : "#333333";
    },

    generateColors(colors, colorAdjust){
      const isVersion1 = this.version === 1,
        name = this.colorName.replace(/\s/gi, "-");

      for (const key in colors) {
        const color = colors[key],
          adjustedColor = colorAdjust(this.color, color),
          label = isVersion1 ? key : `'${name}-${key}'`;

        this.colors.push({
          name: `${name}-${key}`,
          label,
          background: adjustedColor,
          text: this.getTextColor(adjustedColor)
        });
      }
    },

    generate() {
      // Reset the colors
      this.colors = [];

      const isVersion1 = this.version === 1,
        name = this.colorName.replace(/\s/gi, "-");

      // Tints
      this.generateColors(this.tints, this.tint)

      const label = isVersion1 ? "500" : `'${name}'`;

      // Base
      this.colors.push({
        name,
        label,
        background: `#${this.color}`,
        text: this.getTextColor(this.color)
      });

      // Shades
      this.generateColors(this.shades, this.shade)

      this.$track("colors", "generated", `${this.name}:${this.color}`);

      // Notify our parent that we're done
      this.$emit("generate", this.colors);
    },

    rename() {
      if (this.newName === this.colorName) {
        this.isRenaming = false;
        return;
      }

      const isColorAlreadyAdded =
        this.allColors.findIndex(
          c => c.name.toLowerCase() === this.newName.toLowerCase()
        ) > -1;

      if (isColorAlreadyAdded) {
        this.$emit("color:duplicated", {
          name: this.newName,
          color: this.color
        });
        return;
      }

      this.$store.commit("RENAME_COLOR", {
        currentName: this.colorName,
        newName: this.newName
      });
      this.$track("colors", "renamed", `${this.colorName} to ${this.newName}`);
      this.colorName = this.newName;
      this.generate();
      this.isRenaming = false;
    }
  },

  watch: {
    isRenaming(value) {
      if (!value) return;

      this.newName = this.colorName;

      this.$nextTick(_ => {
        this.$refs.newName.focus();
      });
    }
  },

  computed: {
    allColors() {
      return this.$store.getters.colors;
    },

    text() {
      return this.getTextColor(this.color);
    },

    tints() {
      return this.tintsPerVersion[this.version];
    },

    shades() {
      return this.shadesPerVersion[this.version];
    }
  }
};
</script>

<template>
  <div class="palette">
    <span class="close float-right m-4 cursor-pointer" @click="remove()">
      <i class="far fa-trash-alt text-white"></i>
    </span>
    <ul>
      <li
        :style="{
                    backgroundColor: '#'+color,
                    color: text
                }"
      >
        <div
          v-if="!isRenaming"
          class="cursor-pointer hover:opacity-75"
          @click="isRenaming = true"
        >{{ colorName }}</div>
        <div v-else>
          <input
            class="input xs"
            autofocus
            ref="newName"
            type="text"
            @keyup.enter="rename()"
            @keyup.esc="isRenaming = false"
            v-model="newName"
          >
        </div>
        <div class="color-code">#{{ color }}</div>
      </li>
      <!-- Tints and shades -->
      <li
        v-for="(color, $index) in colors"
        :style="{
                    backgroundColor: color.background,
                    color: color.text
                }"
        :key="$index"
      >
        <span>{{ color.label | unquoted }}</span>
        <span class="color-code float-right">{{ color.background.toUpperCase() }}</span>
      </li>
    </ul>
  </div>
</template>
